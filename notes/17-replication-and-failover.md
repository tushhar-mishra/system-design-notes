# 17 · Replication and Failover

> **Level:** ★★★ advanced
>
> **You will learn:** why we copy data to several machines, **leader-follower** (primary-replica), **multi-leader** and **leaderless** replication, **synchronous vs asynchronous vs semi-synchronous** replication and what each means for data loss, **replication lag** and the bugs it causes (read-your-writes), **read replicas**, **automatic failover** step by step, **split brain** and **fencing**, and how managed services (RDS Multi-AZ, Aurora, Cloud SQL HA, Azure zone-redundant HA) implement it.
>
> **You need:** topics 08, 16.
>
> **Part 3 · Caching and data** · Previous: [16 · Database fundamentals](16-database-fundamentals.md) · Next: [18 · Partitioning and sharding](18-partitioning-and-sharding.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Replication** = keeping copies of the same data on several machines. It gives **availability** (a copy takes over if one dies), **read scale** (spread reads over copies), and **lower latency** (copies near users).

```
            writes                    WAL / binlog stream
App ─────────────▶ [Primary] ════════════════════▶ [Replica AZ-b] (standby, sync)
App ── reads ───────────────────────────────────▶ [Replica AZ-c] (read replica, async)
```

- **Synchronous:** primary waits for the replica to confirm before acknowledging the commit → **no data loss** on failover, slower writes.
- **Asynchronous:** primary acknowledges immediately → fast, but the replica **lags** and a failover can **lose the last seconds** of writes.

**Picture it.** A teacher dictating to two note-takers. Synchronous: the teacher waits until the backup note-taker has written each sentence. Asynchronous: the teacher keeps talking; if the main note-taker faints, the backup may be a few sentences behind.

**The one thing to know.** **Replication is not backup.** A `DROP TABLE` or corrupted data replicates to every copy in milliseconds. You need backups/PITR too (topic 41).

**Try it now (Postgres):** `SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;` on the primary.

**You've got it if you can:** explain sync vs async in terms of RPO, describe a failover, and explain split brain and how fencing prevents it.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **Leader / primary / master:** accepts writes.
- **Follower / replica / standby / secondary:** receives changes from the leader. A **standby** waits to take over; a **read replica** serves read traffic (can be both).
- **Replication log:** the stream of changes (Postgres WAL, MySQL binlog).
- **Replication lag:** how far behind a replica is (seconds or bytes).
- **Failover:** promoting a replica to primary when the primary fails. **Switchover:** planned, graceful version (for maintenance).
- **RPO** (Recovery Point Objective) = how much data (time) you can afford to lose. **RTO** (Recovery Time Objective) = how long you can be down. (Full story in topic 41.)

---

## Why it exists

1. **Availability:** machines and AZs fail. A standby in another AZ lets you recover in ~1–2 minutes instead of hours of restore.
2. **Durability:** data on several machines survives a disk/host loss.
3. **Read scaling:** many apps are 10:1 or 100:1 read-heavy; replicas multiply read capacity.
4. **Geography:** replicas in other regions for DR and local reads (topic 42).
5. **Isolation:** analytics/reporting on a replica so it doesn't slow the primary.

---

## Analogy

The teacher and note-takers (Core). Extras:
- **Read replicas** = students copying from the note-takers so they don't all crowd the teacher.
- **Split brain** = the teacher steps out, comes back, and doesn't realise a note-taker has been promoted to teacher; now two people are dictating different things to the class.
- **Fencing** = taking the old teacher's chalk away before the new one starts.

---

## How it works inside

### Leader-follower replication flow

```
1. App: BEGIN; UPDATE accounts ...; COMMIT
2. Primary writes the change to its WAL on disk
3a. ASYNC: primary returns "COMMIT OK" now; WAL streams to replicas shortly after
3b. SYNC:  primary waits until ≥1 sync standby confirms it received (or flushed/applied) the WAL, then returns OK
4. Replica replays the WAL → its data catches up
```

| Mode | Write latency | Data loss on primary failure | Availability risk |
|---|---|---|---|
| **Async** | Lowest | Up to the replication lag (usually < 1 s, can be minutes under load) | None from replication |
| **Sync** (to one standby) | + one round trip to standby (~1 ms cross-AZ) | **Zero** (for confirmed commits) | If the sync standby is down, writes block (unless the DB falls back to async) |
| **Semi-sync / quorum** (e.g. any 1 of 2 standbys; Aurora: 4 of 6 storage copies) | Moderate | Zero for confirmed commits | Tolerates one slow/failed copy |

**Practical pattern:** **synchronous within a region across AZs** (1–2 ms, RPO 0) + **asynchronous across regions** (RPO seconds) because cross-region sync adds 50–200 ms to every commit (topic 03).

### How managed services do it

| Service | How |
|---|---|
| **RDS Multi-AZ (instance)** | Sync block-level replication to a standby in another AZ; standby **not readable**; failover ~60–120 s via DNS endpoint flip |
| **RDS Multi-AZ DB cluster** | One writer + **two readable** standbys in different AZs, semi-sync; faster failover (~35 s or less) |
| **Aurora** | Storage layer keeps 6 copies across 3 AZs (write quorum 4/6, read 3/6); up to 15 read replicas sharing storage (lag usually < 100 ms); failover ~30 s or less; Global Database for cross-region (async, typically < 1 s lag) |
| **GCP Cloud SQL HA** | Regional persistent disk with sync replication to a standby zone; read replicas separate |
| **Azure Database for PostgreSQL Flexible (zone-redundant HA)** | Sync standby in another zone |
| **Self-hosted Postgres** | Streaming replication + **Patroni** (uses etcd/Consul for leader election, topic 26) or a K8s operator |

### Replication lag bugs

With async read replicas, a user can:
- **Not see their own write** (write to primary, read from lagging replica): "I updated my profile but it shows the old name!"
- **See time go backwards** (one read from a fresh replica, next from a lagging one).

Fixes:
- **Read-your-writes:** read from the primary for a short time after a user writes (e.g. 5 s, tracked in session), or for that user's own data.
- **Monotonic reads:** pin a user to one replica.
- **Wait for position:** the app records the primary's log position after a write and asks the replica to wait until it reaches that position (some DBs/proxies support this).
- Send **critical reads** (checkout, balance) to the primary always.

### Failover step by step (automatic)

```
1. Detect:  health checks / consensus fail on primary for N seconds (e.g. 3 × 10 s)
2. Decide:  a failover manager (RDS control plane, Patroni with etcd, Orchestrator) 
            confirms primary is really gone (avoid false positives)
3. Choose:  most up-to-date replica (least lag); with sync standby, it has everything
4. Fence:   make sure the old primary can't accept writes (STONITH = "shoot the other node in the head":
            stop instance, revoke storage, remove from network, or leader lease expiry)
5. Promote: replica becomes primary (opens for writes)
6. Redirect: update DNS endpoint / proxy / Service so apps reconnect to the new primary
7. Rebuild: other replicas follow the new primary; old primary is rebuilt as a replica later
```

Apps see: connection errors for ~30–120 s, then must **reconnect** (connection pools must drop dead connections; DNS TTLs must be short; JVM DNS caching ⚠️). Proxies like RDS Proxy cut client-visible failover time by holding client connections.

### Split brain

Two nodes both believe they're primary and accept writes → **diverging data**, very hard to merge. Causes: a network partition where the old primary is alive but isolated from the failover manager.

Prevention:
- **Quorum-based decisions:** only promote if a **majority** of voters agree (topic 26).
- **Fencing:** guarantee the old primary stops writing (power off, storage fencing, lease expiry: the primary must renew a lease in etcd to keep writing).
- **Fencing tokens:** every new leader gets a higher number; storage rejects writes with old numbers.

### Multi-leader and leaderless

- **Multi-leader:** several nodes accept writes (e.g. one per region), replicate to each other asynchronously. Low local write latency per region; but **write conflicts** (same row changed in two regions) need resolution: last-write-wins (loses data), merge functions, or CRDTs (Conflict-free Replicated Data Types: data structures designed to merge automatically). Used by some multi-region setups (DynamoDB global tables use last-writer-wins; also MySQL group replication in multi-primary mode, Postgres BDR).
- **Leaderless** (Dynamo-style: Cassandra, Riak, DynamoDB internally): the client writes to several replicas and reads from several; **quorums** decide (topic 19). Tolerates node failures without failover, but consistency is tunable and eventual.

---

## Options and trade-offs

| Topology | Writes | Consistency | Failover | Choose when |
|---|---|---|---|---|
| Single leader + async replicas | One node | Strong on primary, stale on replicas | Needed; RPO > 0 | Most read-heavy apps |
| Single leader + sync standby | One node | Strong; RPO 0 | Needed; fast | Production OLTP default (Multi-AZ) |
| Multi-leader (multi-region) | Many | Eventual, conflicts | Not needed per region | Multi-region writes with conflict-tolerant data |
| Leaderless quorum | Any node | Tunable | Not needed | Huge write volume, AP systems |

| Read replica usage | Pros | Cons |
|---|---|---|
| Send all reads to replicas | Max offload | Lag bugs everywhere |
| Send only tolerant reads (lists, search, reports) | Safe | Less offload |
| Cache first, replicas second | Biggest win usually | Two layers of staleness |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Primary dies (async)** | Failover; last N seconds of writes lost | Failover event; gap reconciliation | Sync standby in-region; accept documented RPO |
| **Replication lag spike** | Stale reads, users confused; failover would lose more | `ReplicaLag` alert (e.g. > 30 s) | Find cause: heavy write batch, long query on replica, under-sized replica; throttle batch jobs |
| **Replica falls too far behind / breaks** | Replica stops; reads fail over to primary → primary overload | Replica state | Rebuild; size replicas like the primary |
| **Split brain** | Two primaries, data divergence | Two writable nodes | Quorum + fencing; never "manually promote" without fencing |
| **App doesn't reconnect after failover** | Errors continue after DB is back | Errors persist past failover end | Short DNS TTL, client DNS cache settings, pool validation, RDS Proxy |
| **Sync standby down → writes block** | Write latency/timeouts | Sync state | Quorum sync over 2 standbys, or auto fallback to async with alert |
| **Replicating a mistake** | `DELETE` without `WHERE` reaches all replicas | Data checks | PITR backups, delayed replica (applies changes 1 h late) for quick recovery |
| **Failover flapping** | Repeated failovers | Event history | Tune detection thresholds, investigate root cause before failback |

**3 AM:** "Writes failing, RDS event says failover in progress." Expect ~1–2 min. Watch: app reconnect errors, connection pool behaviour. After: check for lost writes if async (compare business events), check that the old primary is fenced, and whether replicas re-attached.

---

## Where you'll use it

- **Every production DB design**: say "Multi-AZ, sync standby, async read replicas for X, cross-region replica for DR".
- **Kubernetes:** etcd uses Raft (leader + followers + quorum) ([K8s 77 · etcd backup and DR](../../CKA-Certification-Course-2025/notes/77-etcd-backup-restore-and-dr.md)).
- **Jenkins/CI:** stateful controllers need replication or fast restore (topic 47).
- **Terraform:** `multi_az = true`, `aws_db_instance` with `replicate_source_db` for read replicas.
- **Case study 52:** DR for a critical database builds on this topic.

---

## Design exercise

**Prompt:** A payments service on Postgres: 2,000 writes/s, 15,000 reads/s. Requirements: RPO 0 for AZ failure, RPO ≤ 1 min for region failure, RTO ≤ 5 min (AZ) and ≤ 1 h (region). Users must always see their own payments right after making them. Design the replication.

<details>
<summary><b>Model answer</b></summary>

- **In-region:** Aurora PostgreSQL (or RDS Multi-AZ DB cluster): writer in AZ-a; storage/standbys across 3 AZs with quorum writes → **RPO 0** for an AZ failure; automatic failover ~30 s → meets RTO 5 min.
- **Reads:** 2–3 reader instances behind the reader endpoint. Route **only lag-tolerant reads** there (history lists older than a few seconds, reports). **Read-your-writes:** after a user creates a payment, their reads go to the writer for 10 s (flag in session), and payment status checks always go to the writer.
- **Cross-region:** Aurora Global Database (async, typically < 1 s lag) to a secondary region → **RPO ≈ seconds** (well under 1 min). Managed/unplanned failover promotes the secondary in minutes; plus app stack warm in the DR region (topic 41–42) → RTO < 1 h.
- **Not replication only:** PITR backups (35 days) + daily snapshots copied cross-region; a "delayed" safety net for logical mistakes via PITR.
- **Apps:** RDS Proxy or short DNS TTL + connection validation; retries with idempotency keys for payment writes (topic 06/22) so a failover mid-write doesn't double-charge.
- **Monitoring:** replica lag, Global DB lag (alert > 30 s, since RPO is 1 min), failover events, connection errors.
</details>

---

## Quick recap

- Replication = copies for availability, read scale, locality; it is **not** backup.
- Sync = RPO 0, slower writes; async = fast, RPO = lag. Sync in-region across AZs, async across regions.
- Replica lag causes read-your-writes bugs: route critical/own reads to the primary.
- Failover: detect → decide (quorum) → fence → promote → redirect → rebuild. Apps must reconnect.
- Split brain is prevented by quorum and fencing.

### Interview questions

1. **Q: Your RDS failover took 90 seconds but the app was broken for 10 minutes. Why?**
   A: The app didn't reconnect promptly: connection pools held dead connections, the JVM cached the old DNS answer, or retries without backoff overwhelmed the new primary. Fix: short DNS TTL and client DNS cache, pool connection validation/max lifetime, RDS Proxy, and retry with backoff and idempotency.

2. **Q: Why not use synchronous replication to another region for RPO 0?**
   A: Every commit would wait for a cross-region round trip (tens to hundreds of ms), cutting write throughput and raising latency, and a region link problem would block writes. Most designs accept a small RPO cross-region with async replication, keeping sync replication inside the region. If true RPO 0 across regions is required, use a system built for it (e.g. Spanner-style consensus across regions) and accept the latency.

3. **Q: How do you prevent split brain in a self-managed Postgres cluster?**
   A: Use a consensus-backed manager like Patroni with etcd: the leader holds a lease in etcd and must renew it; a node only acts as primary while it holds the lease, and promotion requires the distributed lock. Add fencing (the old primary demotes itself when it loses the lease; watchdog reboots it). Never promote manually without confirming the old primary is fenced.
