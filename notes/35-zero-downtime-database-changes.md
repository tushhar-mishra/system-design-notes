# 35 · Database Changes Without Downtime

> **Level:** ★★★ advanced
>
> **You will learn:** why schema changes are the riskiest part of a deploy, the **expand/contract** (parallel change) pattern step by step, **backward- and forward-compatible** migrations, which DDL operations **lock** tables (and how to avoid it: `CREATE INDEX CONCURRENTLY`, `lock_timeout`, online schema change tools like **gh-ost / pt-online-schema-change**), **backfilling** big tables safely, running **migrations in CI/CD** (as a separate step, not on app startup), **rollback** of data changes, and the failure modes that cause outages.
>
> **You need:** topics 16, 17, 33, 34.
>
> **Part 6 · Delivery** · Previous: [34 · Deployment strategies](34-deployment-strategies.md) · Next: [36 · IaC at scale](36-iac-at-scale.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** During any rolling/canary deploy, **old and new code run at the same time against the same database**. So every schema change must work with **both** versions. The pattern is **expand → migrate → contract**:

```
Rename column "name" → "full_name" without downtime:
1. EXPAND:   add column full_name (nullable)                     ← old code ignores it
2. DEPLOY:   code v2 writes BOTH name and full_name, reads name
3. BACKFILL: copy name → full_name for old rows (in batches)
4. DEPLOY:   code v3 reads full_name (still writes both)
5. DEPLOY:   code v4 stops writing name
6. CONTRACT: drop column name (after v4 is stable, days later)
```

**Picture it.** Replacing a bridge while traffic keeps flowing: build the new bridge next to the old one (expand), route some cars, then all cars over it (migrate), and only demolish the old bridge when nobody uses it (contract).

**The one thing to know.** **Never combine a destructive schema change with the code change that needs it.** Additive first, destructive last, each in a separate deploy. Then rollback of code is always safe.

**Try it now (Postgres):** `SET lock_timeout = '3s';` before any DDL in a migration, so a blocked `ALTER` fails fast instead of queueing every other query behind it.

**You've got it if you can:** plan a column rename/type change in steps, explain why `ALTER TABLE` can cause an outage, and design a safe backfill.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **Migration:** a versioned script that changes schema or data (Flyway, Liquibase, Alembic, Rails/Django migrations, golang-migrate, Atlas).
- **DDL** (Data Definition Language): `CREATE/ALTER/DROP`. **DML** (Data Manipulation Language): `INSERT/UPDATE/DELETE`.
- **Backward compatible:** the new schema works with the **old** code (needed during rollout and rollback).
- **Forward compatible:** the old schema works with the **new** code (needed if code deploys before migration, or for blue/green).
- **Expand/contract (parallel change):** add new structure, migrate usage, remove old structure later.
- **Backfill:** a data migration filling new columns/tables for existing rows.
- **Online schema change:** changing big tables without long locks (copy to a shadow table and swap).

---

## Why it exists

- Code can be rolled back in seconds; **data and schema can't** (a dropped column is gone).
- Some DDL takes **exclusive locks**: on a busy table, an `ALTER TABLE` waiting for a lock blocks every subsequent query on that table → instant outage.
- Big data migrations (updating 500 M rows) in one transaction can bloat the DB, lag replicas, fill disks and hold locks for hours.
- With canaries and rolling deploys (topic 34), mixed versions are guaranteed.

---

## Analogy

Bridge replacement (Core). Also: changing the format of a shared paper form in an office. You can't swap all forms overnight; first accept both old and new forms, then switch everyone to filling the new form, then stop accepting the old one.

---

## How it works inside

### Which changes are safe?

| Change | Safe online? | How |
|---|---|---|
| Add nullable column (no default, or constant default in Postgres 11+/MySQL 8 instant) | ✅ | Plain `ADD COLUMN` (metadata-only) |
| Add column with volatile default / NOT NULL on big table | ⚠️ | Add nullable → backfill → add constraint `NOT VALID` → `VALIDATE` |
| Create index | ⚠️ locks writes by default | Postgres `CREATE INDEX CONCURRENTLY`; MySQL InnoDB online DDL / gh-ost |
| Add foreign key | ⚠️ | `ADD CONSTRAINT ... NOT VALID` then `VALIDATE CONSTRAINT` (weaker lock) |
| Rename column/table | ❌ breaks old code | Expand/contract (new column + dual write) or a view |
| Change column type | ❌ often rewrites table | New column + backfill + switch + drop |
| Drop column/table | ❌ breaks old code | Only after no code version uses it (contract phase), and after backup |
| Add NOT NULL | ⚠️ full scan | `CHECK (col IS NOT NULL) NOT VALID` → validate → set NOT NULL (PG 12+ uses the check) |

### The lock queue trap

```
t0  Long SELECT (analytics, 2 min) holds a shared lock on orders
t1  Migration: ALTER TABLE orders ADD COLUMN x ... → needs ACCESS EXCLUSIVE → WAITS for t0
t2  Every new query on orders → queues BEHIND the waiting ALTER  → app threads pile up → outage
```
Even a "fast" metadata-only ALTER can cause this. Defence:
```sql
SET lock_timeout = '3s';          -- give up quickly if we can't get the lock
SET statement_timeout = '15min';  -- for long operations like CONCURRENTLY builds
-- retry the migration a few times with backoff
```

### Expand/contract, full example (change `price` from INTEGER cents to NUMERIC with currency)

```
Release 1 (expand):   ADD COLUMN amount NUMERIC(12,2) NULL, ADD COLUMN currency CHAR(3) NULL
Release 2 (dual write): code writes price AND amount/currency; reads price
Backfill job:          UPDATE products SET amount = price/100.0, currency='INR'
                       WHERE id BETWEEN :a AND :b AND amount IS NULL   -- batches of 5,000, sleep between
Release 3 (switch reads): code reads amount/currency (fallback to price if NULL during transition)
Verify:                 count(*) WHERE amount IS NULL = 0 ; consistency checks
Release 4 (stop old writes): code no longer writes price
Release 5 (contract):   (days/weeks later, after backup) DROP COLUMN price
```
Each release is independently deployable and **rollback-safe** to the previous one.

### Safe backfills

- **Batches** by primary key range (1,000–10,000 rows), each its own short transaction.
- **Throttle:** sleep between batches; watch replica lag and DB CPU; pause automatically if lag > threshold.
- **Idempotent and resumable:** `WHERE new_col IS NULL`; track progress.
- Run as a **separate job** (K8s Job / worker), not inside the app's startup or the migration transaction.
- Off-peak for very large ones; and estimate time: 500 M rows at 5,000 rows/batch × 50 ms → 100,000 batches × (50 ms + 50 ms sleep) ≈ 2.8 h.

### Online schema change tools (MySQL mainly)

```
gh-ost / pt-online-schema-change:
1. Create shadow table _orders_gho with new schema
2. Copy rows in chunks
3. Capture ongoing changes (binlog stream for gh-ost, triggers for pt-osc) and apply to shadow
4. When caught up: brief lock, atomic rename swap
```
Postgres: most changes via the patterns above; tools like `pg_repack` (rebuild tables without long locks) and logical replication for big rewrites.

### Migrations in the pipeline

```
CI: lint migrations (e.g. squawk for Postgres, Atlas lint: detect locking/destructive ops)
    → test migrations against a copy of prod schema (and size-realistic data for big tables)
CD: run migrations as a SEPARATE step BEFORE the code that needs them:
    K8s Job / Argo CD PreSync hook / pipeline stage, with one runner at a time (migration lock table)
    → then deploy code (canary)
Destructive (contract) migrations: separate PR, separate release, after verification window
```
⚠️ Don't run migrations on app startup in every pod: 50 pods racing, slow startups failing readiness, half-applied migrations during crashes.

### Rollback of data changes

- Prefer **roll forward** with a new migration over "down" migrations on production data.
- Keep **PITR** backups (topic 41); take an extra snapshot before risky migrations.
- Destructive steps last + a waiting period = you rarely need a data rollback.

---

## Options and trade-offs

| Approach | Pros | Cons |
|---|---|---|
| Maintenance window with downtime | Simple | Downtime; not acceptable for 24/7 services |
| Expand/contract | Zero downtime, rollback-safe | More releases, temporary dual-write code |
| Online schema change tools | Big table changes without long locks | Extra disk (copy of table), load, tooling complexity |
| Blue/green with separate DBs | Isolated | Data sync between DBs is hard; rarely worth it |
| Views/compat layers | Hide renames | Complexity, performance traps |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **DDL waiting on lock → queue** | Sudden total outage on one table | Blocked queries, lock waits | `lock_timeout`, retry; kill long transactions before migrations |
| **Index build locks writes** | Writes time out | Lock monitoring | `CONCURRENTLY` / online DDL |
| **Destructive change with code** | Old pods crash during rollout; rollback fails | Errors mid-rollout | Expand/contract; lint for drops/renames |
| **Huge single-transaction backfill** | Replica lag, bloat, disk full, long locks | Lag, WAL growth | Batched, throttled, resumable backfills |
| **Migration ran twice / concurrently** | Errors, partial state | Migration logs | Migration tool's lock table, single job |
| **`CONCURRENTLY` index failed** | Invalid index left behind | `pg_index.indisvalid = false` | Drop and recreate |
| **Migration fine in staging, slow in prod** | Hours-long lock in prod | — | Test against prod-sized data; estimate time |
| **Forgot contract phase** | Schema clutter, dual-write bugs | Audit | Track contract tasks with dates |

**3 AM:** "Orders table: everything timing out right after the deploy." Check for a migration waiting on a lock (`pg_stat_activity` where `wait_event_type = 'Lock'`). Cancel the migration (`pg_cancel_backend`) → queue drains → service recovers. Retry later with `lock_timeout` after killing the long-running query that blocked it.

---

## Where you'll use it

- Every **zero-downtime deployment** design (case study 48).
- **Pipeline stages** for migrations in Jenkins/GHA, Argo CD PreSync hooks.
- **Platform tools upgrades** (SonarQube, Grafana, Keycloak) run their own migrations: snapshot first (topic 41).
- **Terraform** has the same idea for infrastructure: `moved`, `create_before_destroy` ([Terraform 24 · lifecycle](../../Terraform/notes/24-lifecycle.md), [Terraform 40](../../Terraform/notes/40-moved-and-removed.md)).

---

## Design exercise

**Prompt:** Split the `users.address` free-text column into a new `addresses` table (one user, many addresses) for a 200 M-row table with 5,000 writes/s, without downtime.

<details>
<summary><b>Model answer</b></summary>

1. **Expand:** create `addresses(id, user_id FK NOT VALID → validate later, line1, city, ..., is_primary)` and index `addresses(user_id)` (created on an empty table, so instant).
2. **Release A (dual write):** on every user address update, write to `users.address` **and** upsert the primary row in `addresses`, in the same transaction. Reads still from `users.address`.
3. **Backfill job:** batches of 5,000 users by ID range: parse `address` → insert into `addresses` where no primary address exists (idempotent). Throttle on replica lag (< 5 s) and DB CPU (< 60%). Estimated: 40,000 batches × ~150 ms ≈ 1.7 h of work, run over a day with throttling.
4. **Verify:** counts and sampled comparisons between `users.address` and `addresses` primary rows; fix parser edge cases; rerun idempotently.
5. **Release B (switch reads):** read from `addresses` (feature flag, ramp 1% → 100%), still dual-writing.
6. **Release C (multiple addresses feature):** now possible on the new table.
7. **Release D:** stop writing `users.address`.
8. **Contract (2–4 weeks later):** snapshot, then `ALTER TABLE users DROP COLUMN address` with `lock_timeout`.

Each step is rollback-safe: until D, the old column is complete and authoritative-compatible.
</details>

---

## Quick recap

- Old and new code share the DB during deploys: every migration must be backward compatible.
- Expand → dual write → backfill → switch reads → stop old writes → contract (later, after backup).
- Avoid locks: `lock_timeout`, `CREATE INDEX CONCURRENTLY`, `NOT VALID` + `VALIDATE`, online schema change tools.
- Backfills: batched, throttled on replica lag, idempotent, resumable, separate jobs.
- Run migrations as a separate pipeline step before code; roll forward; destructive changes last.

### Interview questions

1. **Q: How do you rename a column with zero downtime?**
   A: Expand/contract: add the new column, deploy code that writes both and reads the old, backfill in batches, deploy code that reads the new, stop writing the old, and finally drop the old column in a later release. Each step is independently deployable and safe to roll back.

2. **Q: Why can a quick `ALTER TABLE` take down production?**
   A: It needs an exclusive lock. If a long query holds a conflicting lock, the ALTER waits, and every new query on that table queues behind it, so the app's threads pile up and the service stalls. Set a short `lock_timeout` with retries and avoid long-running transactions around migrations.

3. **Q: Should the app run migrations on startup?**
   A: Not in production at scale. Many replicas race, startups slow down and fail probes, and crashes can leave partial migrations. Run migrations as a dedicated, single-runner step (Job, PreSync hook, pipeline stage) before deploying the code that needs them, with linting and prod-sized testing.
