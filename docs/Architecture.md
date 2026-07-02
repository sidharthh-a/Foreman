# Foreman — Distributed Job Scheduling Platform
## Architecture Blueprint v1.0

---

## 1. Executive Summary

Foreman is a multi-tenant, horizontally scalable job scheduling platform that lets projects define queues, submit jobs (immediate, delayed, scheduled/cron, batch), and execute them reliably across a fleet of workers. The system's defining constraint is **exactly-once claiming, at-least-once execution**: no job may be claimed by two workers, but a worker crash must never silently lose a job.

The architecture is split into four independently deployable services — **API Service**, **Scheduler Service**, **Worker Service**, and **Dashboard (frontend)** — coordinated through **PostgreSQL** (system of record) and **Redis** (coordination, queue signaling, pub/sub for live updates). Each service has a single responsibility and can scale independently. Complexity is intentionally kept low: no message broker beyond Redis, no external workflow engine, no microservice sprawl — just enough separation to make each piece independently scalable, testable, and replaceable.

The design prioritizes correctness of the job lifecycle and observability over feature breadth, in line with the assignment's evaluation criteria.

---

## 2. High-Level Architecture

```
                         ┌─────────────────────┐
                         │   React Dashboard    │
                         │ (REST + WebSocket)   │
                         └──────────┬───────────┘
                                    │
                          HTTPS / WSS
                                    │
                         ┌──────────▼───────────┐
                         │      API Service       │
                         │  (FastAPI, stateless)  │
                         │  Auth, CRUD, Job intake │
                         │  WebSocket gateway      │
                         └───┬────────────┬───────┘
                             │            │
                 writes/reads│            │pub/sub events
                             │            │
                 ┌───────────▼──┐   ┌─────▼──────┐
                 │  PostgreSQL   │   │   Redis     │
                 │ (system of    │   │ (queue      │
                 │  record)      │   │ signaling,  │
                 │               │   │ pub/sub,    │
                 │               │   │ locks)      │
                 └───────▲───────┘   └──────▲──────┘
                         │                  │
             ┌───────────┴─────┐    ┌───────┴────────┐
             │ Scheduler Service │    │  Worker Fleet   │
             │ (APScheduler,     │    │ (N replicas,    │
             │  single leader)   │    │  polling +      │
             │  promotes due     │    │  heartbeats)    │
             │  jobs → queued    │    │                 │
             └───────────────────┘    └─────────────────┘
```

Each box is a separate container/deployment unit. API and Worker scale horizontally by adding replicas; Scheduler runs as a leader-elected singleton (logic, not data, must not duplicate); PostgreSQL and Redis are shared, centrally managed dependencies.

---

## 3. Major Components and Responsibilities

**API Service (FastAPI)**
- Sole owner of authentication (JWT issuance/validation) and authorization (RBAC).
- All CRUD for Organizations, Projects, Queues, Retry Policies.
- Job submission endpoint — validates payload, writes job row, does **not** execute anything.
- Read APIs for jobs, executions, logs, worker status, metrics — with pagination and filtering.
- WebSocket gateway: subscribes to Redis pub/sub channels and fans updates out to connected dashboard clients. Does not generate the events itself.
- Never touches job execution logic. Single responsibility: request/response and data access boundary.

**Scheduler Service (APScheduler-based)**
- Owns time: evaluates delayed jobs, cron/recurring definitions, and batch expansion.
- Runs one active instance (leader election via a Redis/Postgres advisory lock) to avoid duplicate promotion of the same due job. Standby instances stay warm for failover.
- Its only write operation is: move a job from `scheduled` to `queued` state and push a signal to Redis. It never claims or executes jobs.
- Computes next run time for recurring jobs after each promotion.

**Worker Service (horizontally scaled pool)**
- Polls assigned queues (or subscribes to Redis signals to avoid busy polling), atomically claims one job at a time per execution slot using a `SELECT ... FOR UPDATE SKIP LOCKED`-style claim.
- Executes job payloads inside a bounded concurrency pool (per-worker concurrency limit from queue config).
- Emits heartbeats on an interval to prove liveness; a job whose worker stops heartbeating is reclaimed.
- Applies the job's retry policy on failure, or routes to the Dead Letter Queue when retries are exhausted.
- Writes execution logs/metrics incrementally, not just at completion, so partial progress is observable.
- Stateless beyond in-flight jobs — can be killed and restarted without data loss (in-flight jobs simply time out and get reclaimed).

**Dashboard (React SPA)**
- Pure presentation layer. Reads via REST, subscribes via WebSocket for live queue/worker/job state.
- No business logic beyond client-side filtering/formatting; all authority lives server-side.

**PostgreSQL**
- System of record for every durable entity: users, orgs, projects, queues, jobs, executions, logs, workers, heartbeats, retry policies, DLQ entries.
- Source of truth for atomic job claiming (via row-level locking), not Redis — this avoids a split-brain between two coordination stores.

**Redis**
- Coordination and notification layer, not a source of truth: queue "ready" signals (to avoid pure polling), scheduler leader lock, pub/sub channel for dashboard live updates, optional rate-limiting counters.
- Everything in Redis is reconstructable from Postgres if Redis is flushed — this is a deliberate resilience boundary.

---

## 4. Service Interaction Flow

**Job submission (immediate job):**
1. Dashboard/API client → API Service: `POST /jobs` with queue id, payload, priority.
2. API validates project/queue ownership and quota, inserts a `jobs` row with status `queued`.
3. API publishes a lightweight "queue X has work" signal to Redis.
4. API returns `202 Accepted` with job id immediately — submission is decoupled from execution.

**Job submission (delayed/scheduled/recurring):**
1. Same API entry point, but status is set to `scheduled` with a `run_at` timestamp or cron expression stored.
2. Scheduler Service picks it up when due, flips status to `queued`, signals Redis.

**Worker claim and execution:**
1. Worker receives (or polls for) a Redis signal for a queue it services.
2. Worker executes an atomic claim query against Postgres: find the highest-priority `queued` job in that queue not already claimed, lock the row, set status `claimed` + `worker_id` + `claimed_at` in one transaction.
3. Worker transitions the job to `running`, begins heartbeating, executes the payload.
4. On success: status → `completed`, execution record finalized, results stored.
5. On failure: retry policy evaluated — either re-queued with backoff delay or moved to `failed`/DLQ.
6. Each transition publishes an event to Redis pub/sub → API's WebSocket gateway → Dashboard.

**Dashboard live view:**
1. Dashboard opens a WebSocket to the API.
2. API subscribes that connection to relevant Redis channels (per project/queue).
3. State changes stream to the browser without polling; REST is used for initial page load and historical queries.

---

## 5. Job Lifecycle

```
        submit
          │
          ▼
     ┌─────────┐   run_at due    ┌─────────┐
     │Scheduled│ ───────────────▶│  Queued  │◀────────────────┐
     └─────────┘                 └────┬────┘                  │
    (delayed/cron                     │ worker claims          │ retry
     jobs only;                       ▼                        │ (delay elapsed)
     immediate jobs                ┌─────────┐                 │
     skip this state)              │ Claimed │                 │
                                    └────┬────┘                 │
                                         ▼                       │
                                    ┌─────────┐   failure &      │
                                    │ Running │──retries left────┘
                                    └────┬────┘
                          success        │        failure & retries exhausted
                           ┌─────────────┴───────────────┐
                           ▼                              ▼
                     ┌───────────┐                 ┌─────────────┐
                     │ Completed │                 │ Dead Letter  │
                     └───────────┘                 │    Queue     │
                                                     └─────────────┘
```

Notes:
- `Claimed` is a distinct, short-lived state from `Running` so a worker crash between claim and start-of-execution is visibly distinguishable and reclaimable via heartbeat timeout.
- Recurring jobs don't have a single lifecycle instance — each firing creates a new `job execution` row against a persistent `scheduled job` definition, keeping history queryable per occurrence.
- A job manually retried from the DLQ re-enters at `Queued`, with retry count reset and an audit trail entry linking it to the original failure.

---

## 6. Worker Lifecycle

1. **Startup**: worker registers itself in the `workers` table (id, hostname, capacity, assigned queues, status = `starting`).
2. **Ready**: status → `active`; begins heartbeat loop (e.g., every 5–10s) and starts polling/subscribing.
3. **Claiming loop**: attempts atomic claims up to its configured concurrency limit; idles (or blocks on Redis signal) when no work or at capacity.
4. **Executing**: runs job(s) in its concurrency pool; heartbeats continue independently of job execution so a hung job doesn't starve liveness signaling.
5. **Heartbeat monitor** (server-side, run by Scheduler or a lightweight reaper job): any worker whose last heartbeat exceeds a threshold is marked `unresponsive`; jobs it held in `claimed`/`running` are reclaimed and re-queued (or failed if idempotency can't be guaranteed and retries are exhausted).
6. **Graceful shutdown**: on SIGTERM, worker stops accepting new claims, finishes or checkpoints in-flight jobs within a drain timeout, marks itself `draining` then `stopped`, deregisters.
7. **Crash (no graceful path)**: no special handling required beyond the heartbeat monitor — the system is designed to treat "crashed" and "slow" identically, which keeps recovery logic simple.

---

## 7. Queue Processing Flow

1. Each queue has configuration: priority tier, max concurrency, retry policy reference, paused flag.
2. **Signaling, not busy-polling**: job submission publishes to a Redis channel keyed by queue id; workers subscribed to that queue wake up rather than tight-loop querying Postgres. A low-frequency fallback poll (e.g., every N seconds) covers missed signals — this is the simplicity/robustness trade-off: Redis pub/sub is a hint, Postgres state is the truth.
3. **Priority**: within a queue, claim query orders by `priority DESC, created_at ASC` to guarantee FIFO within the same priority band.
4. **Concurrency limiting**: enforced at claim time — a worker (or the pool of workers servicing that queue) will not claim beyond the queue's configured concurrent-execution ceiling; the claim query filters on a live count of `running` jobs for that queue.
5. **Pause/resume**: paused queues are simply excluded from the claim query and from scheduler promotion; in-flight jobs are unaffected, only new claims stop.
6. **Batch jobs**: submitted as a single logical batch record that expands into N individual job rows at submission time (not at claim time), each independently claimable, with the batch's own status derived (aggregated) from its children.
7. **Backpressure**: if a queue's backlog exceeds a threshold, the API can reject new submissions with `429` or flag the queue as "under pressure" for dashboard visibility — a simple, explicit form of the "rate limiting" bonus feature.

---

## 8. Database Overview (Tables Only)

| Table | Purpose |
|---|---|
| `users` | Authentication identity and profile |
| `organizations` | Top-level tenant boundary |
| `organization_members` | User ↔ Organization membership + role |
| `projects` | Owned by an organization; owns queues |
| `project_members` | Fine-grained project-level RBAC (optional layer) |
| `queues` | Queue configuration: priority, concurrency limit, paused flag |
| `retry_policies` | Reusable retry strategy definitions (fixed/linear/exponential) |
| `jobs` | Core job record: type, payload, status, priority, queue reference |
| `scheduled_jobs` | Delayed/cron/recurring definitions that generate `jobs` rows |
| `job_executions` | One row per execution attempt of a job (supports retry history) |
| `job_logs` | Structured log lines/events tied to a job execution |
| `workers` | Registered worker instances, capacity, status |
| `worker_heartbeats` | Time-series liveness pings per worker |
| `dead_letter_entries` | Permanently failed jobs, with failure context and replay linkage |
| `job_batches` | Groups related jobs submitted together; aggregates child status |
| `audit_logs` | Cross-cutting record of privileged actions (retries, pauses, RBAC changes) |

Schema-level detail (columns, indexes, FKs) is intentionally deferred to the design-decisions/database design phase, per assignment scope.

---

## 9. Deployment Architecture

- **Containerization**: each service (API, Scheduler, Worker, Dashboard) ships as its own Docker image with its own Dockerfile; a `docker-compose.yml` orchestrates local/dev environments including Postgres and Redis.
- **Horizontal scaling units**:
  - API: stateless, scale by replica count behind a load balancer; WebSocket connections require sticky routing or a Redis-backed pub/sub fan-out so any API replica can serve any subscriber (already the design — no extra work needed).
  - Worker: scale by replica count; queue assignment can be uniform (all workers poll all queues) or partitioned (worker pools dedicated to specific queues) for isolation between noisy and critical workloads.
  - Scheduler: exactly one active leader via advisory lock; additional replicas are hot standbys, not extra capacity.
- **Single-responsibility deployability**: any service can be redeployed, restarted, or scaled independently without touching the others — this is the practical payoff of the service split.
- **Environment separation**: dev/staging/prod isolated at the Postgres/Redis instance level, with environment-specific config injected via env vars (never baked into images).
- **Migrations**: Alembic migrations run as a discrete pre-deploy step (job/init-container), never implicitly on service boot, to avoid multiple replicas racing to migrate.
- **Reverse proxy / TLS termination**: sits in front of API and Dashboard; not part of Foreman's own services.
- **Observability sidecar concerns** (logging/metrics shipping) are attached at the infra level, not built into application logic beyond structured log output.

---

## 10. Recommended Development Order

1. **Foundations**: Postgres schema + Alembic migrations for core tables (users, orgs, projects, queues) → Auth (JWT) → basic project/queue CRUD API.
2. **Job intake**: `jobs` table, job submission endpoint (immediate jobs only first), job read/list endpoints with pagination/filtering.
3. **Worker MVP**: single-instance worker that polls, atomically claims, executes a trivial job type, marks completed/failed — proves the atomic-claim mechanism before anything else is layered on.
4. **Retry & DLQ**: retry policies, execution history, dead letter queue, manual retry-from-DLQ.
5. **Scheduling**: delayed jobs, then cron/recurring via the Scheduler service and leader election.
6. **Heartbeats & reclaim**: worker heartbeat table, reaper logic for stuck/crashed workers — this closes the reliability loop.
7. **Concurrency & priority controls**: per-queue concurrency limits, priority ordering, pause/resume.
8. **Batch jobs**: batch submission and status aggregation.
9. **Real-time layer**: Redis pub/sub wiring, WebSocket gateway in API, signaling replacing pure polling in workers.
10. **Dashboard**: read views first (queues, jobs, workers) → live updates via WebSocket → operational actions (retry, pause, cancel).
11. **Hardening**: structured logging, metrics, rate limiting/backpressure, RBAC refinement, tests, documentation.
12. **Bonus features** (only after core is solid): workflow dependencies, distributed locking primitives beyond claim, queue sharding, event-driven triggers, AI-generated failure summaries.

This order front-loads the two hardest correctness problems — atomic claiming and failure recovery — before any UI or convenience feature, matching the evaluation's emphasis on reliability over feature count.

---

## 11. Architectural Risks and Mitigation

| Risk | Mitigation |
|---|---|
| **Duplicate job execution** (two workers claim the same job) | All claims go through a single atomic, row-locking transaction in Postgres (`FOR UPDATE SKIP LOCKED` pattern); Redis is never the source of truth for claim state. |
| **Lost jobs on worker crash** | Heartbeat-based reclaim: any job whose owning worker stops heartbeating past a threshold is returned to `queued` (or failed, if execution isn't idempotent and retries are exhausted). |
| **Scheduler duplicating promotions** (two scheduler instances both promote the same due job) | Leader election via advisory lock; only the elected instance promotes `scheduled → queued`. |
| **Thundering herd on queue signal** | Redis pub/sub signal is a wake-up hint, not a work item; the actual atomic claim query naturally serializes contention at the database. |
| **Non-idempotent job execution retried and double-applying side effects** | Encourage/require idempotency keys on job payloads for job types with external side effects; document this as a contract for job authors rather than something the platform can fully guarantee. |
| **Hot queue starving others** | Per-queue concurrency limits plus priority tiers prevent one queue's backlog from monopolizing shared worker capacity. |
| **Database becoming the bottleneck at scale** | Claim query is indexed and short-lived; heavier read traffic (dashboards, logs) is served from indexes designed for filtering, and can later be offloaded to a read replica without touching write-path logic. |
| **WebSocket fan-out complexity as API scales horizontally** | Redis pub/sub decouples "who generated the event" from "who is connected where" — any API replica can serve any subscriber without direct coordination. |
| **Retry storms after outages** | Backoff strategies (linear/exponential) are mandatory on retry policies, not optional, and DLQ acts as a circuit breaker for jobs that will never succeed. |
| **Schema churn as job types diversify** | Job `payload` and `result` are stored as structured (JSONB) fields rather than rigid typed columns, avoiding migrations per new job type while keeping core lifecycle columns strongly typed. |

---

## 12. Enterprise Best Practices

- **Single responsibility per service**: API never executes jobs; Scheduler never claims jobs; Workers never handle auth or CRUD. This makes failure domains and scaling decisions independent and easy to reason about.
- **Database as source of truth, cache/broker as coordination aid**: Redis loss should degrade performance (more polling), never correctness (no lost or duplicated jobs).
- **Idempotent, atomic state transitions**: every lifecycle transition is a single transaction with explicit pre-conditions (e.g., "claim only if still `queued`"), preventing race conditions under concurrent workers.
- **Observability by default**: every execution attempt, not just the final outcome, produces logs and metrics — this is what makes the dashboard's "job explorer" and throughput visualizations meaningful rather than cosmetic.
- **Structured, consistent error handling**: API errors follow a single schema (code, message, field errors) across all endpoints; internal exceptions are never leaked to clients.
- **Least-privilege RBAC**: roles scoped at organization and project level, validated server-side on every request — the dashboard's role-based UI is a convenience, not a security boundary.
- **Config over code**: retry policies, concurrency limits, and queue behavior are data-driven (rows in Postgres), not hardcoded per job type, so operational tuning doesn't require redeployment.
- **Migrations as a first-class, reviewed artifact**: Alembic migrations are version-controlled and applied as an explicit deploy step, never inferred from ORM models at runtime.
- **Graceful degradation over premature optimization**: sharding, distributed locking beyond the claim mechanism, and event-driven execution are explicitly deferred to the bonus phase — the core system must be correct and simple before it is elaborate.
- **Documentation as a deliverable, not an afterthought**: architecture diagram, ER diagram, API docs, and a design-decisions document (trade-offs, not just choices) are treated as required outputs of the engineering process, not optional polish.

---

*This document defines the architectural contract for Foreman. Detailed database schemas (columns, indexes, constraints), API contracts, and component-level design should be produced as follow-on artifacts that conform to the boundaries and lifecycles defined here.*
