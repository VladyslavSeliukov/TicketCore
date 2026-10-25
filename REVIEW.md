# TicketCore — Senior Review v2 (single source of truth)

> Claim set: README "Deep Dive" sections + benchmark table. The resume bullets were not provided; re-run section 2 once they are.
> Method: full read of all tracked files. Nothing was executed except read-only checks: grep, git log, inspecting the installed `arq` source, and comparing the locust secret with `.env`.
> Origin tags: **[v1]** found only by v1 and confirmed; **[v2-new]** missed by v1; **[v1+v2]** found independently by both, or v1's finding materially corrected or extended by v2. ⚠️ = interview risk, 📄 = resume/README-claim mismatch.

## 1. Overall verdict
**Not resume-ready, at Junior+ or Middle.** Two bugs break the core flow at runtime:
- The worker talks to the wrong Redis database, so expired reservations never return to the Redis gatekeeper and the "GC reconciliation" is a no-op (TC-001).
- `DELETE /tickets/{id}` double-refunds inventory, which enables real overselling past `tickets_quantity` (TC-002).

On top of that, four README claims are false or unmeasured: `SKIP LOCKED`, "strictly-once", the benchmark table, and "90% offload". Fix the MUST-FIX list (about 3–4 focused days) and this becomes a solid Junior+ project with a genuinely good Middle-level talking point: the Redis↔Postgres dual-write.

## 2. Resume / README claims

| # | Claim | Status | Evidence |
|---|---|---|---|
| 1 | O(1) cache invalidation via versioned keys + `INCR` | **Implemented** (minor race) | `src/core/redis_keys.py:46-80`. GET-then-SET on a missing version key (`:59-63`); see TC-023. |
| 2 | Redis Lua "gatekeeper" absorbs flash-sale load, rejects sold-out in memory | **Partial / misleading** 📄 | `src/services/ticket.py:62-88`. The counter drifts in both directions (TC-001, TC-006), and the warm-up path races. With the seeded 5,000-ticket type it filters nothing until sold out. The real guards are the DB `CHECK` (`src/models/ticket_type.py:45-47`) and a hot-row lock (TC-010). |
| 3 | `@idempotent` "hashes payloads **and headers**", "strictly-once execution" | **Misleading** 📄 | Headers are not hashed (`src/api/decorators.py:32-42` takes only Pydantic models and primitives). The header is optional (`:75`). TTL is 10 s (`.env.example:28`). Unlock after commit (`:103-107`). See TC-005. |
| 4 | GC cron sweeps Redis reservations vs Postgres with 60 s grace | **Implemented in code, broken at runtime** 📄 | `src/worker.py:174-228` reads the hash from Redis **db 1**, but the API writes it to **db 0**. See TC-001. |
| 5 | "`SELECT … FOR UPDATE SKIP LOCKED` in booking and payment" | **Not implemented** 📄 | Zero occurrences. Booking uses an atomic `UPDATE` (`ticket.py:91-96`); payment and worker use plain `with_for_update()` (`payment.py:51-56`, `worker.py:76`). See TC-003. |
| 6 | Read-aside cache offloads "90%+" of `GET /events` | **Implemented; number unmeasured** 📄 | `src/repositories/event.py:113-197`. No hit/miss metric. Locust never calls the list endpoint (`locustfile.py:56-63`). |
| 7 | Terraform ephemeral EKS, in-cluster Postgres/Redis, Prometheus + Grafana | **Partial** 📄 | TF VPC/EKS/ECR exist. k8s has no worker Deployment, no PVCs, an unused PgBouncer, and no Prometheus or Grafana (compose only). `:latest` vs an IMMUTABLE ECR repo. See TC-011, TC-012. |
| 8 | ~2,500 peak / 1,500 avg RPS, 0.00% errors, oversell 0, idempotency failures 0 | **Misleading** 📄 | `load_tests/locustfile.py`: 409 is counted as success, no Idempotency-Key, forged JWTs, no DB assertions. 8 workers claimed vs `--workers 1` in compose. The report predates 15 commits. See TC-004. |

## 3. Findings

### Critical

**TC-001 — Worker reads and writes inventory in the wrong Redis database** [v2-new] ⚠️📄
- Category: correctness / distributed state.
- Location: `src/worker.py:72,131,181` use `ctx["redis"]`; `src/worker.py:234-236` sets `RedisSettings(database=1)`. The API side uses `src/db/redis.py:24-28` with `REDIS_URL` → `/0` (`src/core/config.py:55`).
- Root cause: arq injects its own queue connection as `ctx['redis']` (verified in the installed `arq.worker`: `self.ctx['redis'] = self.pool`). All inventory keys, `system:active_reservations` and `system:canceled_tickets` are written by the API to db 0. The worker's `refund_inventory_idempotent` runs against db 1, where the inventory key doesn't exist, so the Lua `EXISTS` check skips the `INCR`.
- Effects:
  1. Every timed-out reservation is cancelled in Postgres, but its slot never returns to the Redis counter. The API then answers "Sold out" while the DB has free capacity: permanent undersell until the key is reset.
  2. `reconcile_redis_orphans` reads an empty hash in db 1 and never does anything.
  3. The db 0 reservation hash grows forever.
- Why it's hidden: every worker test uses `AsyncMock()` for Redis (`tests/integration/workers/test_garbage_collector.py:49` and others), so they only assert the call arguments.
- Fix: in `startup`, create a dedicated client from `settings.REDIS_URL` (`ctx["app_redis"]`) and use it in all three tasks. Add one integration test with a real Redis that reserves, expires and asserts the counter.

**TC-002 — `TicketService.delete` double-refunds inventory, which enables real oversell** [v1+v2] ⚠️
- Category: concurrency / data integrity.
- Location: `src/services/ticket.py:192-243`.
- Root cause: no status check, no row lock, hard delete. Deleting a ticket that is already CANCELED (refunded by the worker, `worker.py:91-98`) decrements `tickets_sold` again, and the Redis `INCR` (`:226-232`) bypasses the idempotent canceled-set path.
- Effects:
  - `tickets_sold` undercounts, so the DB `CHECK (tickets_quantity >= tickets_sold)` admits more sales than capacity. This is a **true oversell**, the one thing the project exists to prevent. At 0 it instead fails `tickets_sold >= 0` with a 500.
  - SOLD tickets can be deleted, freeing inventory with no payment trace.
  - With no `FOR UPDATE`, a user delete racing the worker's cancel decrements twice even for a RESERVED ticket.
- Fix: `SELECT … FOR UPDATE`; allow only RESERVED; transition to CANCELED instead of deleting (one cancel path shared with the worker); refund Redis via `refund_inventory_idempotent`. Add a regression test: worker cancels, then user deletes, then assert `tickets_sold` is unchanged.

**TC-003 — README claims `SKIP LOCKED`; it doesn't exist** [v1] ⚠️📄
- Category: resume integrity.
- Location: README §5 "Strict Pessimistic Locking" vs all of `src/`.
- Why it matters: anyone can check it with `grep` in 10 seconds, and it taints every other claim. `SKIP LOCKED` would also be *wrong* for payment, because a skipped row looks like "not found".
- Fix: state the truth: "atomic conditional `UPDATE` + CHECK constraint for booking; `FOR UPDATE` on the ticket row for the pay/expire race". Be ready to explain where `SKIP LOCKED` does fit (queue consumers).

**TC-004 — The benchmark table isn't backed by the test** [v1+v2] ⚠️📄
- Category: resume integrity / testing.
- Location: `load_tests/locustfile.py`, README "Benchmark Results".
- Root cause, from v1:
  - 409 "sold out" is counted as success (`:82-91`).
  - No Idempotency-Key is sent, so "idempotency failures 0" is vacuous.
  - Nothing computes an oversell index.
  - `quantity` and `event_id` are sent but ignored by `TicketCreate`.
  - The README claims `--workers 8`; compose runs 1 (`docker-compose.yml:34`).
  - With 8 workers, `prometheus_client` without multiprocess mode would report per-process garbage.
- Added by v2:
  - Locust **forges JWTs with the server's real `SECRET_KEY`** (`:24`, `:48-54`), so login and Argon2 (TC-009) were never under load.
  - It POSTs to `/api/v1/tickets/` while the route is `/api/v1/tickets` (`src/main.py:76-78` + `tickets.py:21`). Starlette's slash redirect then likely adds a 307 hop per purchase (PLAUSIBLE, not executed).
  - The only committed report is dated 2026-03-11, before "Worker update (#29)" and 14 later commits.
- Fix: re-run against current code with idempotency keys and deliberate retries. Afterwards, assert in SQL `tickets_sold <= tickets_quantity` and `count(tickets WHERE status<>'CANCELED') = tickets_sold`, and check that the Redis counter equals the remainder. Report p50/p95/p99 plus "local M1, single instance". Or delete the table.

**TC-005 — Idempotency is not "strictly-once"** [v1+v2] ⚠️📄
- Category: reliability.
- Location: `src/api/decorators.py:75,90-107`; `src/services/idempotency.py:69-74,126`; `REDIS_TTL_SECONDS=10` (`.env.example:28`, `k8s/01-config.yaml`, CI).
- Root causes (v1):
  1. The header is optional, so omitting it bypasses protection.
  2. A `save_response` failure after commit hits `except` → `unlock`, and a retry re-executes.
  3. IN_PROGRESS and COMPLETED share a **10 s** TTL. A request slower than 10 s (DB `pool_timeout=10`) loses its lock, and any retry after 10 s re-executes.
  4. A crash between commit and save leaves a lock that expires, and the action re-executes.
  5. The user and service are discovered by `isinstance`-scanning kwargs, so a refactor silently disables it.
- Added by v2: `asyncio.CancelledError` is a `BaseException`, so it skips `except Exception` and leaves IN_PROGRESS until the TTL. The README says headers are hashed; they aren't.
- Fix: require the header on `POST /tickets` and payments. TTLs: IN_PROGRESS about 30–60 s, COMPLETED 24 h. Never unlock once side effects committed. The real exactly-once design is a `(user_id, key)` unique row written in the same Postgres transaction as the ticket, with Redis only as a fast path.

### Major

**TC-006 — Redis counter drifts in both directions in `TicketService.create`** [v1+v2] ⚠️
- Category: concurrency / dual-write.
- Location: `src/services/ticket.py:75-139`.
- Severity: v1 had Critical; **changed to Major**, because the DB CHECK still prevents oversell. Impact is undersell and wasted DB load.
- Root causes:
  - (v1) Compensation happens only on `SQLAlchemyError`. A Redis `ConnectionError` on `hset`, a timeout or a cancellation after `DECR` leaks a slot, and nothing ever reconciles the counter against the DB.
  - (v1) `enqueue_job` after commit (`:118`): a failure there gives a 500, the idempotency unlock fires, and the client retry creates a second reservation.
  - (v2) Warm-up race (`:77-85`): N concurrent requests all see `-1`, all read the DB and all `SET available`, each overwriting the others' `DECR`s. So after a Redis restart or a missing key, the gate admits far more than capacity.
  - (v2) **Double refund**: when the commit fails, the code `INCR`s (`:136-138`), but the `hset` reservation entry (`:102-110`) was written before the commit and stays. Once TC-001 is fixed, `reconcile_redis_orphans` sees `db_status is None` and refunds **again**. v1 said "the GC cleans it"; it actually double-counts.
  - (v2) On a CHECK violation (DB is genuinely full), the compensation `INCR`s (`:125`), inflating a counter that was already wrong.
- Fix:
  - Warm with `SET NX` (or do get-or-init inside one Lua script).
  - Compensate in `except BaseException`.
  - Write the hash entry after commit (or delete it on rollback).
  - Make the enqueue best-effort, since the cron is the safety net.
  - Add a periodic reconcile `counter := quantity - sold - in-flight`.
  - Use `register_script`/EVALSHA.
- Interview angle: "the DB is the source of truth, Redis is an optimistic filter".

**TC-007 — Payment accepted after the reservation expired** [v1]
- Location: `src/services/payment.py:51-70`.
- Root cause: only the status is checked. A RESERVED ticket past `TICKET_RESERVATION_TIME_SECONDS` stays payable until the deferred job or the 5-minute cron runs (or forever on k8s, see TC-011).
- Fix: check `created_at + ttl < now()` inside the locked transaction, and cancel there.

**TC-008 — "Payment" is a status flip** [v1] 📄
- Location: `src/services/payment.py`.
- Root cause: there is no amount, provider, payment record, refund or webhook, yet the README sells double-charge prevention.
- Fix: either call it "confirm purchase (mock PSP)", or add a `payments` table with unique `(ticket_id)`, a provider reference and a status machine. The current branch `feat/banking` suggests this is in progress.

**TC-009 — Argon2 hashing blocks the event loop** [v2-new] ⚠️
- Category: async correctness / performance.
- Location: `src/services/auth.py:41,74,107,111` → `src/core/security.py:24,38`.
- Root cause: argon2-cffi defaults (64 MiB, t=3) cost tens of ms of CPU per call, run synchronously in `async def`. Every login, signup or password change stalls all in-flight requests on that worker. The benchmark never exposed this because it forged tokens (TC-004).
- Fix: `await run_in_threadpool(ph.verify, …)` (or `anyio.to_thread.run_sync`), plus login rate limiting (TC-015).

**TC-010 — Hot-row lock held across network I/O serialises all bookings of a ticket type** [v2-new] ⚠️
- Category: concurrency / scalability.
- Location: `src/services/ticket.py:91-112`.
- Root cause: `UPDATE ticket_types SET tickets_sold = tickets_sold + 1` takes the row lock. Then the transaction does an INSERT flush and **a Redis `HSET` round-trip**, and only then commits. Every reservation for that type queues on one row lock for the whole span. The Redis gate only filters requests once the type is sold out, so before sell-out this row lock caps throughput.
- Fix:
  - Keep the transaction to `UPDATE` + `INSERT` + commit; do the Redis work outside it.
  - Know the alternatives: derive "sold" from counting tickets with a periodic rollup, sharded counters, or a queue-based allocator.
  - Be able to explain why the CHECK still makes this safe.

**TC-011 — On k8s, reservations never expire** [v1+v2] ⚠️📄
- Location: `k8s/` has no worker Deployment; `k8s/07-postgres.yaml`, `k8s/08-redis.yaml`.
- Severity: v1 had "C7, Critical" as a bundle; **re-scoped to Major** for this part.
- Root cause: there is no arq worker Deployment, so neither `release_unpaid_ticket` nor the crons ever run. Postgres and Redis are Deployments with no PVC, so a pod restart wipes the data. A Redis restart also triggers the TC-006 warm-up race.
- Fix: add a worker Deployment (with liveness via the 8001 metrics port). Add PVCs, or say "ephemeral demo, data loss acceptable" in the README.

**TC-012 — PgBouncer is deployed but unused; pool math doesn't close** [v1+v2]
- Location: `k8s/01-config.yaml` (`POSTGRES_HOST: "postgres-svc"`), `k8s/09-10-pgbouncer-*.yaml`, `src/db/session.py:12-26`, `k8s/05-backend-hpa.yaml`.
- Root cause:
  - v1 couldn't tell whether the backend used PgBouncer. **Verified: it doesn't**, so PgBouncer is dead infra.
  - HPA max 10 × (10 + 10 overflow) = 200 connections against stock Postgres `max_connections=100`, so it exhausts under scale-out.
  - If PgBouncer were wired in, transaction mode plus asyncpg needs `statement_cache_size=0` / `prepared_statement_cache_size=0`, and the `jit=off` startup parameter must be added to `IGNORE_STARTUP_PARAMETERS` (latent).
- Fix: point the app at `pgbouncer-svc` with the asyncpg settings above, or delete PgBouncer. Document the connection budget.

**TC-013 — Real `SECRET_KEY` committed in the load test** [v1+v2] ⚠️
- Category: security.
- Location: `load_tests/locustfile.py:24`; in history since `4f65269`.
- Root cause: v2 verified it **equals the key in the current local `.env`**. Anyone with repo access can mint a token for any user ID, including the superuser's (`sub` is just the user ID).
- Fix: rotate it now and read it from an env var in locust. No history rewrite is needed once rotated.

**TC-014 — No per-user limit, no rate limiting** [v1]
- Root cause: one account can loop `POST /tickets` and hold the entire inventory for 15 minutes, again and again. That is the scalping threat a flash-sale system exists to stop. `/auth/login` is unthrottled (brute force, plus the CPU DoS via TC-009).
- Fix: a per-user cap per ticket type (DB partial unique index or count check), and rate limiting at nginx or in the app.

**TC-015 — Ticket-type admin update races the counter; bad FK becomes a 500** [v1]
- Location: `src/services/ticket_type.py:184-207`, `:49-58`.
- Root cause: `redis.set(quantity - tickets_sold)` after commit overwrites in-flight `DECR`s. Creating a type with a nonexistent `event_id` raises an FK `IntegrityError` that surfaces as a 500, not a 404/422.
- Fix: adjust by delta with `INCRBY`, and validate the event exists (or map the `IntegrityError`).

**TC-016 — Missing indexes** [v1]
- Location: `migrations/versions/3ecc93ee2e17_initial_schema.py:63-74`.
- Root cause: there are no indexes on `tickets.owner_id`, `tickets.ticket_type_id` or `ticket_types.event_id` (Postgres doesn't auto-index FKs), and no `(status, created_at)` index for the GC `UPDATE` (`worker.py:138-144`). `ix_tickets_status` alone has low selectivity.
- Fix: a new migration with those indexes, ideally a partial index `WHERE status='RESERVED'`.

**TC-017 — Tests miss every path that broke** [v1+v2] ⚠️
- From v1:
  - No payment tests at all.
  - No `IdempotencyService` or decorator tests (replay, conflict, in-progress).
  - No Lua tests, and no inactive or expired token tests.
  - Tests share one `AsyncSession` with the app.
  - `flushdb()` runs on the configured Redis (`tests/conftest.py:103-107`).
- Added by v2:
  - The worker tests mock Redis, which is exactly why TC-001 survived.
  - The stress test (`tests/stress/test_concurrency.py`) creates the ticket type directly in the DB, so Redis has no counter. All 50 requests take the racy `-1` warm-up path (TC-006) and the **DB CHECK** produces the 5/45 split. It proves the constraint, not the Lua gate.
  - The `ticket_factory` fixture (`tests/integration/conftest.py:103-115`) calls `TicketFactory.build(event_id, …)` positionally, so it is broken and unused.
  - The custom `event_loop` fixture is deprecated under pytest-asyncio ≥ 1.0. v1 called it "dead"; it is still requested by `setup_test_db`, so it is deprecated rather than dead.
- Fix: add tests for TC-001/002/005/007 with real Redis, and a stress variant that pre-warms the counter and runs multiple processes.

**TC-018 — CI runs only pytest** [v1+v2]
- Location: `.github/workflows/ci.yml:73-77`.
- Root cause (v1): no ruff, no mypy despite strict config, no `alembic upgrade head` (tests use `create_all`, so migration drift and index names are untested), no Docker build, no coverage.
- Added by v2: `uv sync` without `--frozen`; triggers only for `main`; leftover locust env vars (`:42-43`).
- Fix: add jobs for lint, typecheck, `alembic upgrade head && alembic check`, and `docker build`.

### Minor

**TC-019 — Worker robustness** [v1+v2]
- Location: `src/worker.py`.
- Details:
  - The Redis refund runs **before** `commit` (`:100-104`). On commit failure, Redis is refunded and marked processed while the DB is still RESERVED (v1).
  - `queue_monitor_task` is never cancelled in `shutdown` (`:41`, `:46-53`).
  - A failing `release_unpaid_ticket` is not retried: arq retries only on `Retry`, so recovery waits for the cron (v2).
  - `GRACE_PERIOD_SECONDS` is a local constant.
- Fix: commit first, then refund idempotently. Cancel the monitor task. Raise `Retry(defer=…)` on transient errors.

**TC-020 — Unbounded Redis structures** [v1+v2]
- Root cause: `system:canceled_tickets` (`redis_keys.py:93`) has no TTL and grows by one entry per cancellation, forever. `system:active_reservations` in db 0 is never cleaned for expired tickets (TC-001).
- Fix: per-ticket keys with a TTL, or periodic trimming.

**TC-021 — `is_active` not enforced after login** [v1]
- Severity: v1 had Critical; **changed to Minor**. No endpoint can deactivate a user (`UserUpdate` is unused), so the bug is latent.
- Fix: check `is_active` in `get_current_user` (`src/api/deps.py:83-89`). It's one line.

**TC-022 — Register race returns 500** [v1]
- Severity: v1 had Major; **changed to Minor**, since it only fires on a rare race.
- Location: `src/services/auth.py:35-56`.
- Root cause: check-then-insert. The unique index raises `IntegrityError`, which surfaces as a 500. Failed-login logs contain the email (PII, `:75`).

**TC-023 — Cache edge cases** [v1]
- Redis down → `GET /events` returns 500, with no DB fallback.
- No stampede protection.
- Version key GET-then-SET (`redis_keys.py:59-63`). Low impact, since noeviction is the default.
- `TypeAdapter` is rebuilt on every call (`repositories/event.py:173`).

**TC-024 — Pagination without `ORDER BY`** [v1]
- Severity: v1 had Major; **changed to Minor**.
- Location: `repositories/event.py:76`, `services/ticket.py:181-186`, `services/ticket_type.py:105-110`.
- Fix: add `ORDER BY id`, or use keyset pagination.

**TC-025 — Observability gaps** [v1+v2]
- JSON logs are enabled only when `ENVIRONMENT == "prod"` (`src/core/logger.py:92`), but k8s sets `"production"`, so they are never JSON.
- No request ID or tracing, and no cache hit/miss metric.
- The readiness failure is logged at INFO.
- (v2) `/metrics` is publicly exposed through nginx and the k8s LoadBalancer (`src/main.py:67-72`).
- The global `SQLAlchemyError` handler logs `str(exc)`, which includes SQL parameters.

**TC-026 — Event delete semantics** [v2-new]
- Location: `services/event.py:96-116`, `repositories/event.py:90-94`.
- Root cause: `ticket_types.event_id ON DELETE RESTRICT` blocks deleting any event that has ticket types, even with zero tickets, yet the message says "existing tickets". The `IntegrityError` → `EventDeleteError` mapping is duplicated in the repo and the service, and the service branch is unreachable.

**TC-027 — Docker/compose hygiene** [v1+v2]
- Severity: v1 had Major; **changed to Minor**.
- From v1:
  - `uv:latest` and `prom/prometheus:latest` are unpinned.
  - `curl` is installed but there is no `HEALTHCHECK`.
  - `PYTHONPATH=/app/src` is wrong.
  - The `.:/app` bind mount overrides the image.
  - Migrations run in the app start command, so replicas race.
  - Prometheus scrapes a `cadvisor` that compose doesn't define.
- Added by v2:
  - `.dockerignore` contains only `*.mp4`, so the build context ships `.env` and `.venv`.
  - `UV_COMPILE_BYTECODE` is set in the runtime stage, where it does nothing.
  - Postgres `max_connections=1000` is set in compose, with no reasoning.

**TC-028 — k8s/Terraform odds and ends** [v2-new]
- ECR is `IMMUTABLE` (`terraform/aws/ecr.tf:3`) but the manifests use `:latest`, so a second push of `latest` fails.
- Alertmanager mounts the secret `ticketcore-secrets` while everything else uses `ticketcore-secret`.
- There is no Prometheus and there are no alert rules in k8s, so Alertmanager has nothing feeding it.
- A LoadBalancer Service plus an Ingress is redundant.
- `t3.micro` nodes (about 4 pods per node) vs HPA max 10 + PDB `minAvailable: 2`.
- The AWS account ID is hardcoded.

**TC-029 — LLM-generated signals and dead code** [v1+v2] ⚠️
- Google-style `Args/Returns` docstrings on one-liners everywhere.
- The `RedisClient = Redis` alias is repeated in 5 modules.
- `UserUpdate` is unused.
- `idempotency_service` parameters exist only so the decorator can find them.
- `TicketTypeQuantity` lacks the `Error` suffix; the `busines.py` filename is a typo.
- Redundant `Field(ge=0)` on `NonNegativeInt`; `IdempotencyRecord.status: str` instead of a `Literal`.
- `# type: ignore[misc]` on every `eval`; Lua inline in three places.
- The README duplicates the "Background Jobs" line and has the wrong `cd ticket-core`.
- The "Eventer" typo in `create_superuser.py`.
- `tests/create_test_superuser.py` hardcodes a personal email.
- Inconsistent patterns: a repository layer for Event only, and hard delete vs soft CANCELED.

## 4. Rejected v1 findings
- **"Crons run on every worker replica uncoordinated."** Wrong: `arq.cron(unique=True)` is the default (verified signature), so one execution per tick across workers.
- **"`start_http_server(8001)` raises on restart in the same process."** Speculative: arq doesn't re-run `on_startup` in-process.
- **"`get_redis` builds a `Redis` per request and never closes it" (leak).** Wrong: `Redis(connection_pool=shared)` doesn't own the pool, so nothing leaks.
- **"Confirm `terraform.tfstate` never entered history."** Verified: it never did. Non-issue.
- **"Telegram `chat_id` committed" as a security item.** A chat ID is not a credential; at most it's cosmetic.
- **"PgBouncer will reject `jit`."** Moot today, because the app bypasses PgBouncer. Folded into TC-012 as latent.

## 5. Top 10 interview questions (with the trickiest follow-ups)
1. **Walk me through `POST /tickets` end to end.** → The process dies between `DECR` and commit: who restores the slot? → It dies after commit but before `enqueue_job`? (TC-006)
2. **The DB CHECK already prevents oversell. Why Redis?** → What happens when Redis restarts empty mid-sale and 1,000 requests hit the warm-up path at once? (TC-006) → Where's your actual throughput ceiling? (TC-010, the hot row)
3. **Which Redis database does your worker write the refund to?** (TC-001. If they read `worker.py` next to `redis.py`, this is a 30-second kill.) → Why didn't your tests catch it?
4. **Show me `SKIP LOCKED`.** Say it isn't there, first. → When is `SKIP LOCKED` right? Why pessimistic `FOR UPDATE` for payment rather than optimistic versioning?
5. **A user cancels a ticket the worker already expired. What happens to `tickets_sold`?** (TC-002) → What about a sold ticket?
6. **How does idempotency work? The request takes 12 s and your TTL is 10.** → Commit succeeded but Redis `SET` failed? → Why not store the key in Postgres in the same transaction? (TC-005)
7. **How did you measure "oversell 0" and "90% offload"?** → Locust counts 409 as success and forges tokens, so what did you actually measure? p99? DB connections at peak? (TC-004)
8. **Your login is `async def`. What does Argon2 do to the event loop?** (TC-009) → How would you size threadpool vs workers?
9. **Pool math: 10 pods × 20 connections; PgBouncer in transaction mode with asyncpg.** → What breaks with prepared statements? Is your app even going through PgBouncer? (TC-012)
10. **What delivery guarantee does arq give `release_unpaid_ticket`?** → What if it raises: is it retried? → Can a user pay after expiry before the cron runs? (TC-019, TC-007)

## 6. Fix plan

### MUST-FIX (before sending the next application)
1. **TC-001**: give the worker a db-0 Redis client, plus one real-Redis expiry test.
2. **TC-002**: lock the row, allow cancel only from RESERVED, soft-cancel through the shared path, plus a regression test.
3. **TC-003 / TC-004 / TC-005 / claims 2, 4, 6**: make the README true. Remove `SKIP LOCKED`, "strictly-once", "headers", "90%", and the benchmark table (or re-run it per TC-004).
4. **TC-013**: rotate `SECRET_KEY`; locust reads it from env.
5. **TC-005 (minimal)**: required header on booking/payment, 24 h COMPLETED TTL, no unlock after commit.
6. **TC-007**: expiry check inside the payment lock.
7. **TC-009**: Argon2 via threadpool.
8. **Tests**: payment (happy path, double pay, expired) + idempotency replay/conflict + the TC-001/TC-002 tests above.

### BACKLOG
- TC-006 (SET NX warm-up, BaseException compensation, hash after commit, periodic counter reconcile, EVALSHA)
- TC-010 (shrink the transaction)
- TC-011 (worker Deployment, PVCs)
- TC-012 (wire or delete PgBouncer)
- TC-014 (per-user cap, rate limit)
- TC-015
- TC-016 (index migration)
- TC-017 (rest of the test gaps)
- TC-018 (CI lint/mypy/alembic/docker)
- TC-019, TC-020, TC-021, TC-022, TC-023, TC-024, TC-025, TC-026, TC-027, TC-028, TC-029
- TC-008 (real `payments` table, once `feat/banking` lands)
