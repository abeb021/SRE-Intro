# Lab 1 — SRE Philosophy: Deploy, Break, Understand

## Task 1 — Deploy & Break QuickTicket

### 1.1 / 1.2 — Deploy & verify

```
$ docker compose up --build -d
$ docker compose ps
NAME             IMAGE                COMMAND                  SERVICE    STATUS                    PORTS
app-events-1     app-events           "uvicorn main:app --…"   events     Up (healthy)              0.0.0.0:8081->8081/tcp
app-gateway-1    app-gateway          "uvicorn main:app --…"   gateway    Up                        0.0.0.0:3080->8080/tcp
app-payments-1   app-payments         "uvicorn main:app --…"   payments   Up                        0.0.0.0:8082->8082/tcp
app-postgres-1   postgres:17-alpine   "docker-entrypoint.s…"   postgres   Up (healthy)              0.0.0.0:5432->5432/tcp
app-redis-1      redis:7-alpine       "docker-entrypoint.s…"   redis      Up (healthy)              0.0.0.0:6379->6379/tcp
```

All 5 containers up and healthy.

### Critical path (list → reserve → pay)

```
$ curl -s http://localhost:3080/events | python3 -m json.tool
[
    {"id": 1, "name": "Go Conference 2026", "venue": "Main Hall A", "date": "2026-09-15T09:00:00+00:00", "total_tickets": 100, "price_cents": 5000, "available": 100},
    {"id": 4, "name": "Python Workshop", "venue": "Lab 301", "date": "2026-09-22T14:00:00+00:00", "total_tickets": 25, "price_cents": 2000, "available": 25},
    {"id": 2, "name": "SRE Meetup", "venue": "Room 204", "date": "2026-10-01T18:00:00+00:00", "total_tickets": 30, "price_cents": 0, "available": 30},
    {"id": 5, "name": "Kubernetes Deep Dive", "venue": "Auditorium B", "date": "2026-10-10T10:00:00+00:00", "total_tickets": 80, "price_cents": 8000, "available": 80},
    {"id": 3, "name": "Cloud Native Summit", "venue": "Expo Center", "date": "2026-11-20T10:00:00+00:00", "total_tickets": 500, "price_cents": 15000, "available": 500}
]

$ curl -s -X POST http://localhost:3080/events/1/reserve -H "Content-Type: application/json" -d '{"quantity": 1}' | python3 -m json.tool
{
    "reservation_id": "baae64aa-8b7b-4048-930f-5f7c3918fd46",
    "event_id": 1,
    "quantity": 1,
    "total_cents": 5000,
    "expires_in_seconds": 300
}

$ curl -s -X POST http://localhost:3080/reserve/baae64aa-8b7b-4048-930f-5f7c3918fd46/pay | python3 -m json.tool
{
    "order_id": "baae64aa-8b7b-4048-930f-5f7c3918fd46",
    "event_id": 1,
    "quantity": 1,
    "total_cents": 5000,
    "status": "confirmed"
}
```

### Health check (everything healthy)

```
$ curl -s http://localhost:3080/health | python3 -m json.tool
{
    "status": "healthy",
    "checks": {
        "events": "ok",
        "payments": "ok",
        "circuit_payments": "CLOSED"
    }
}
```

### Dependency map

```
gateway → events → postgres
gateway → events → redis
gateway → payments
```

* `gateway` is a pure router/aggregator — it holds no state of its own.
* `events` is the only service that talks to Postgres (event catalog + orders) and Redis (pending reservations, held-ticket counters).
* `payments` is stateless — mock charge endpoint, tunable failure/latency via env vars.
* `gateway /health` only checks `events` and `payments` directly; Postgres/Redis health is reflected indirectly through `events`' own `/health`.

### 1.4 — Systematic failure exploration

For each component, all other services were left running; only the named component was stopped with `docker compose stop <svc>` and restarted with `docker compose start <svc>` afterwards.

#### Kill `payments`

```
$ docker compose stop payments
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:3080/events
HTTP 200
$ curl -s -X POST http://localhost:3080/events/1/reserve -H "Content-Type: application/json" -d '{"quantity": 1}'
{"reservation_id":"f2223a9d-...","event_id":1,"quantity":1,"total_cents":5000,"expires_in_seconds":300}
$ curl -s -X POST http://localhost:3080/reserve/<id>/pay
{"detail":"Payment service unavailable"}   # HTTP 502
$ curl -s http://localhost:3080/health
{"status":"degraded","checks":{"events":"ok","payments":"down","circuit_payments":"CLOSED"}}   # HTTP 503
```

#### Kill `events`

```
$ docker compose stop events
$ curl -s http://localhost:3080/events
{"detail":"Events service unavailable"}   # HTTP 502
$ curl -s -X POST http://localhost:3080/events/1/reserve -H "Content-Type: application/json" -d '{"quantity": 1}'
{"detail":"Events service unavailable"}   # HTTP 502
$ curl -s http://localhost:3080/health
{"status":"degraded","checks":{"events":"down","payments":"ok","circuit_payments":"CLOSED"}}   # HTTP 503
```

#### Kill `redis`

```
$ docker compose stop redis
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:3080/events
HTTP 200
$ curl -s -X POST http://localhost:3080/events/1/reserve -H "Content-Type: application/json" -d '{"quantity": 1}'
{"detail":"Events service timeout"}   # HTTP 504 — events hangs trying to reach redis (DNS lookup of stopped container)
$ curl -s -X POST http://localhost:3080/reserve/<no-id>/pay
{"detail":"Not Found"}   # HTTP 404 — reservation was never written to Redis, so no reservation_id exists
$ curl -s http://localhost:3080/health
{"status":"healthy","checks":{"events":"ok","payments":"ok","circuit_payments":"CLOSED"}}
```

`/health` stayed "healthy" right after the kill because `events`' Redis check is cached for 5s (`_REDIS_CHECK_INTERVAL`). Within that window the listing endpoint (read-only, Postgres-only) keeps working, but `reserve` — which writes to Redis — times out the whole request chain.

#### Kill `postgres`

```
$ docker compose stop postgres
$ curl -s http://localhost:3080/events
{"detail":"Events service unavailable"}   # HTTP 502
$ curl -s -X POST http://localhost:3080/events/1/reserve -H "Content-Type: application/json" -d '{"quantity": 1}'
Internal Server Error   # HTTP 500
$ curl -s http://localhost:3080/health
{"status":"degraded","checks":{"events":"degraded","payments":"ok","circuit_payments":"CLOSED"}}   # HTTP 503
```

### Failure table

| Component Killed | Events List | Reserve | Pay | Health Check | User Impact |
|-------------------|-------------|---------|-----|--------------|-------------|
| payments | 200 OK | 200 OK | 502 "Payment service unavailable" | 503 degraded (`payments: down`) | Browse + reserve still work; checkout blocked. Reservation stays held in Redis (5 min TTL) so the user can retry payment once `payments` is back. |
| events | 502 "Events service unavailable" | 502 "Events service unavailable" | not tested (no reservation possible without events) | 503 degraded (`events: down`) | Whole app effectively down — `events` is the single point of failure for both reads and writes. |
| redis | 200 OK | 504 "Events service timeout" | 404 (no reservation exists) | 200 healthy (stale, cached redis check) | Browsing still works (Postgres-only). Reservations time out — core write path broken, but health check lags ~5s behind reality. |
| postgres | 502 "Events service unavailable" | 500 Internal Server Error | not tested | 503 degraded (`events: degraded`) | App effectively down for both reads and writes; `events` itself is up but every DB-backed query fails. |

### 1.5 — Load generator + payments failure

```
$ ./app/loadgen/run.sh 5 30
QuickTicket Load Generator
Target: http://localhost:3080 | RPS: 5 | Duration: 30s
---
[10s] requests=46 success=46 fail=0 error_rate=0%
[20s] requests=90 success=88 fail=2 error_rate=2.2%
---
Done. total=130 success=126 fail=4 error_rate=3.0%
```

`docker compose stop payments` was run ~8s into the run (mid first window) and `docker compose start payments` ~30s in (near the end). Error rate jumped from **0% → 2.2%** as soon as payments went down, settling at **3.0%** overall by the end of the run. The failures are concentrated in the "full purchase flow" (10% of traffic) — `pay` calls return 502 while `payments` is down; `events` (70% reads) and `reserve` (20%) keep succeeding throughout, which matches the failure-table finding that listing/reserving are unaffected by a `payments` outage.

## GitHub Community

*(Task 3 — to be completed manually: star the course repo + `simple-container-com/api`, follow `@Cre-eD`, `@Naghme98`, `@pierrepicaud`, and 3+ classmates.)*

- **Why stars matter:** starring bookmarks useful projects, signals community trust/popularity, and surfaces your interests on your profile — helpful for discovering tools and for maintainers to gauge adoption.
- **Why following matters:** following classmates/maintainers surfaces their activity (PRs, issues, projects), which helps spot collaboration opportunities and keeps you visible professionally as your network grows.
