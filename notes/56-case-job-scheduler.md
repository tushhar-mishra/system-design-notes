# 56 · Case Study: A Distributed Job Scheduler

> **Level:** ★★★ advanced
>
> **You will practise:** designing a **company-wide scheduler** for cron-style and one-off jobs: requirements, the **scheduler/dispatcher/worker** split, **leader election** and **partitioned scheduling**, finding due jobs efficiently, **at-least-once execution + idempotency**, **retries and backoff**, **timeouts and heartbeats**, **missed runs and catch-up policy**, **concurrency policies**, multi-tenancy, observability, and when to just use Kubernetes CronJobs or a managed scheduler.
>
> **You need:** topics 16, 18, 21–22, 24, 26, 28.
>
> **Part 9 · Case studies** · Previous: [55 · Rate limiter](55-case-rate-limiter-gateway.md) · Next: [57 · Feature flags and config](57-case-feature-flags-config.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Teams run cron jobs on random VMs and Jenkins. Design a reliable distributed job scheduler for the company."

| Question | Assumed answer |
|---|---|
| Job types? | Recurring (cron expressions) and one-off delayed jobs ("run at 18:00 tomorrow"); jobs run containers |
| Scale? | 50,000 job definitions; up to 2,000 jobs due at the same second (e.g. every hour at :00); 5 M runs/day |
| Duration? | Seconds to hours |
| Guarantees? | Each scheduled run starts within 10 s of its time; **no run silently skipped**; duplicates possible but rare (jobs must be idempotent) |
| Features? | Retries with backoff, timeouts, concurrency policy (allow/forbid/replace), time zones, pause/resume, history and logs, alerts on failure |
| Multi-tenant? | Yes: teams, quotas, isolation |
| Where do jobs run? | Kubernetes clusters (existing) |

---

## 2. Estimation

```
5 M runs/day ÷ 10^5 ≈ 50 runs/s average; peak bursts 2,000 due at the top of the hour
Run records: 5 M/day × 1 KB = 5 GB/day → keep 30 days = 150 GB (partition by day, drop old partitions)
Job definitions: 50,000 × 2 KB = 100 MB (tiny)
Dispatch: 2,000 dispatches in ~5 s → 400/s burst → queue-based dispatch, workers/pods start in parallel
Pods: if avg run 2 min, concurrency ≈ 50/s × 120 s = 6,000 running pods (Little's Law) at average → large! 
   → jobs grouped by tenant into worker pools; many short jobs better run in long-lived worker pods than one pod per run
```
**Conclusion:** the design must handle **thundering-herd scheduling peaks**, **reliable dispatch** (no lost runs), and **execution capacity** efficiently.

---

## 3. High-level design

```
API / UI / GitOps (job definitions as code) ─▶ Jobs DB (Postgres): jobs, schedules, next_run_at, runs
                                                    ▲                │
                       ┌────────────────────────────┘                │
                       │                                             ▼
 SCHEDULER (N replicas, partitioned): each owns partitions of jobs (hash(job_id) % P), via leases
   loop every 1 s: SELECT due jobs WHERE partition IN (mine) AND next_run_at <= now() + 2s
                   FOR UPDATE SKIP LOCKED LIMIT 500
                   → INSERT run (job_id, scheduled_time) UNIQUE(job_id, scheduled_time)   ← dedup
                   → UPDATE next_run_at = next cron occurrence
                   → outbox row "RunDue"                                                 ← same transaction
   outbox relay ─▶ Queue (SQS FIFO per tenant group / Kafka topic per priority)
                                                    │
 EXECUTORS: per tenant pools
   (a) K8s Job launcher: creates a K8s Job per run (long/heavy jobs)
   (b) worker pools (long-lived pods pulling from queue) for many short jobs
   heartbeats → runs table (status RUNNING, last_heartbeat); results → SUCCEEDED/FAILED
 REAPER: finds RUNNING runs with stale heartbeats or past timeout → mark failed → retry policy
 Notifier: failures/missed runs → team alerts ; Metrics/logs per run
```

**Flow for a job "every day 02:00 IST":**
1. At 01:59:58 the owning scheduler finds the job due (`next_run_at` = 02:00), inserts run `(job, 2026-11-21T02:00+05:30)` (unique), updates `next_run_at` to the next day, writes outbox `RunDue` — **one transaction**.
2. Relay publishes to the tenant's queue.
3. Executor receives, checks concurrency policy (if `Forbid` and a previous run is RUNNING → mark SKIPPED with reason), launches the container with the run ID as env var (for idempotency).
4. Container runs; heartbeat every 30 s; exits 0 → SUCCEEDED. Non-zero → retry with backoff up to N attempts → FAILED → alert team.

---

## 4. Deep dives

### 4.1 Finding due jobs efficiently
- Index on `(partition, next_run_at)`; poll every second for `next_run_at <= now() + lookahead`.
- `FOR UPDATE SKIP LOCKED` lets multiple schedulers work safely without blocking each other.
- Alternative for very large scale: a **time-bucketed** store (bucket per minute) or a hierarchical **timing wheel** in memory, rebuilt from DB on failover.

### 4.2 Leader election vs partitioning
- Single leader (K8s Lease) is simple but a throughput and failover bottleneck.
- **Partitioned ownership** ✅: P = 64 partitions; schedulers acquire partition leases (DB rows with lease expiry, or etcd); a dead scheduler's partitions are taken over after the lease expires (~10–15 s).
- Correctness doesn't depend on exclusive ownership: the **unique constraint on (job_id, scheduled_time)** prevents double-creating a run even if two schedulers overlap (topic 26, fencing by the database).

### 4.3 Delivery guarantees
- Run creation + outbox in one transaction → a due run is never lost (topic 22).
- Queue delivery is at-least-once → executors check run status before starting (idempotent start: `UPDATE runs SET status='RUNNING' WHERE id=? AND status='PENDING'`).
- Jobs themselves must be idempotent (they receive `RUN_ID` and `SCHEDULED_TIME`); the platform documents this as a contract.

### 4.4 Missed runs and catch-up
If the scheduler was down from 01:00 to 03:00:
- Policy per job: `catchup: all` (run every missed occurrence), `catchup: latest` (run once now), `catchup: none` (skip, record MISSED). Default: latest, with a `startingDeadlineSeconds` (like K8s CronJob) after which a run is MISSED.
- **Never silently skip:** MISSED runs are recorded and alert the owner.

### 4.5 Thundering herd at :00
- Thousands of jobs scheduled at `0 * * * *`. Mitigate: allow `H`/jitter syntax (Jenkins-style hashing: spread "hourly" jobs across the hour unless exact time required), lookahead pre-dispatch of a few seconds, per-tenant concurrency quotas, and pre-warmed worker pools.

### 4.6 Time zones and DST
Store schedules with explicit IANA time zone; compute next occurrence with a tested library; define DST behaviour (skipped hour → run at next valid time; repeated hour → run once).

---

## 5. Scaling
- Schedulers: add replicas; partitions rebalance via leases.
- DB: modest load; runs table partitioned by day (drop old partitions instantly); read replicas for UI/history.
- Queues: partition by tenant/priority; executors autoscale with KEDA on queue depth (topic 12).
- Execution: large jobs on K8s Jobs (Karpenter, Spot for tolerant jobs); short jobs on worker pools to avoid pod start overhead.

---

## 6. Reliability and DR
| Failure | Effect | Mitigation |
|---|---|---|
| Scheduler replica dies | Its partitions paused ~15 s | Lease takeover; lookahead makes it invisible for most jobs |
| DB failover | Scheduling paused ~30 s | Catch-up on recovery (startingDeadline covers it) |
| Queue down | Runs created but not dispatched | Outbox retains them; relay retries |
| Executor/pod crash mid-run | Heartbeat stops | Reaper marks failed → retry |
| Job hangs | Exceeds timeout | Executor kills it; marks TIMED_OUT; retry policy |
| Region loss | Scheduling stops | DR: DB replica in another region + standby scheduler deployment (pilot light) |

**SLOs:** 99.9% of runs start within 10 s of scheduled time; 0 silently skipped runs (MISSED always recorded and alerted).

---

## 7. Security
- Jobs run with per-team ServiceAccounts / IAM roles (no shared powerful identity).
- Job definitions in Git with review (GitOps) or via API with RBAC per team.
- Secrets injected via the standard secrets path (topic 37), never stored in job definitions.
- Audit: who changed which schedule; who triggered manual runs.

---

## 8. Observability
- Metrics: schedule lag (actual start − scheduled time) p50/p99, runs by status, missed runs, retries, queue depth/age, executor capacity, reaper actions.
- Per-job history and logs (link to log platform with `run_id`).
- Alerts to owners: job failed after retries, job missed, job duration anomaly (much longer than usual), job not succeeded within its SLA window ("nightly export not done by 06:00", topic 28).

---

## 9. Cost
- Small control plane (schedulers, DB, queue).
- Execution compute dominates → Spot, worker pools for short jobs, right-sized resource requests, per-team showback.

---

## 10. Trade-offs and build-vs-buy

| Option | When |
|---|---|
| **K8s CronJob** per team | Small scale; but limited (controller checks every ~10 s; missed-run handling via `startingDeadlineSeconds`; no cross-cluster view, weak retries/alerts) |
| **Managed schedulers** (EventBridge Scheduler, Cloud Scheduler, Azure Logic Apps/Functions timers) | Trigger targets at scale with little ops; combine with queues/containers |
| **Workflow engines** (Temporal, Airflow/Argo Workflows for data pipelines) | Multi-step workflows, dependencies, rich retries |
| **Build (this design)** | Company-wide needs not met by the above (multi-tenant quotas, uniform history/alerts, hybrid targets) |

Senior answer: start with managed scheduling + queues + K8s Jobs and standard alerting; build only if requirements truly exceed them.

---

## 11. Follow-up questions

1. **Q: How do you guarantee a job runs exactly once?**
   A: You can't fully guarantee it across failures; the scheduler guarantees at-least-once with dedup at run creation (unique `(job_id, scheduled_time)`) and idempotent start; the job must be idempotent using `RUN_ID`/`SCHEDULED_TIME` (e.g. conditional writes, unique constraints in its own outputs). That gives effectively-once.

2. **Q: What happens if two schedulers think they own the same partition?**
   A: Both may try to create the same run; the unique constraint lets only one succeed, the other gets a conflict and skips. The database acts as the fencing mechanism.

3. **Q: A job is scheduled every minute but takes 5 minutes. What happens?**
   A: Depends on the concurrency policy: `Allow` runs overlap (dangerous unless the job is designed for it), `Forbid` skips new runs while one is running (records SKIPPED), `Replace` cancels the running one. Default to `Forbid` and alert on repeated skips.

4. **Q: How do you migrate cron jobs from 200 VMs?**
   A: Inventory crontabs (Ansible can collect them), containerize jobs, define them as code with owners and alerts, run in parallel with the old cron in "shadow/dry-run" mode where possible, then disable VM crons host by host, tracking a dashboard of migrated jobs.
