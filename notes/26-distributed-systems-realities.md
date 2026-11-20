# 26 · Distributed Systems Realities

> **Level:** ★★★ advanced
>
> **You will learn:** the **fallacies of distributed computing**, why **clocks can't be trusted** (skew, NTP, monotonic vs wall clock) and what to use instead (**logical clocks, version numbers**), **leader election** and **leases**, **consensus** (Raft in plain words, as used by **etcd**), **quorum size** and why clusters are **3 or 5** nodes, **fencing tokens**, **distributed locks** (and why a Redis lock is not a safe mutex), **idempotency and exactly-once** revisited, and how all of this shows up in Kubernetes, Patroni, Vault and Kafka.
>
> **You need:** topics 17, 19, 22–24.
>
> **Part 4 · Reliability foundations** · Previous: [25 · Rate limiting](25-rate-limiting-and-backpressure.md) · Next: [27 · Observability](27-observability.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Once work spans machines, three things you take for granted disappear: a **reliable network**, a **shared clock**, and **knowing whether another machine is alive**. Distributed systems are designed around these gaps.

```
Network:   messages are lost, delayed, duplicated, reordered; partitions happen
Clocks:    each machine's clock drifts; "now" differs by ms to seconds
Liveness:  "no reply" could mean dead, slow, or a network problem — you can't tell which
```

**Consensus** (Raft, Paxos) lets a group of nodes agree on one value / one leader / one ordered log **as long as a majority (quorum) can talk**. etcd uses Raft; Kubernetes stores all cluster state in etcd.

**Picture it.** A committee voting by letters. Letters can be late or lost, and members' watches disagree. Rule: a decision counts only if a **majority** signs it. Two different majorities can't both exist (they'd overlap in at least one member), so the committee never makes two conflicting decisions, even if some members are unreachable.

**The one thing to know.** **Quorum = majority = ⌊N/2⌋ + 1.** 3 nodes tolerate 1 failure; 5 tolerate 2. **Even numbers add cost without adding tolerance** (4 nodes still tolerate only 1). And spread members across **3 failure domains** (AZs): with 2 AZs, losing the "bigger" one loses quorum.

**Try it now:** `etcdctl endpoint status --cluster -w table` on a kubeadm control plane: see the leader, the Raft term, and each member.

**You've got it if you can:** explain why 3 or 5 nodes, explain leases and fencing, and say why you shouldn't trust timestamps for ordering.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### The fallacies of distributed computing

Assumptions engineers wrongly make (from Sun Microsystems engineers in the 1990s):
1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every resilience pattern (topic 24) exists because one of these is false.

### Key terms

| Term | Meaning |
|---|---|
| **Consensus** | Nodes agreeing on a value despite failures (Raft, Paxos, ZAB) |
| **Leader election** | Choosing one node to coordinate (and keeping only one) |
| **Lease** | A time-limited right ("you're leader for 15 s; renew or lose it") |
| **Term / epoch** | A number that increases with each new leader; used to reject old leaders |
| **Fencing token** | Monotonic number attached to each lock/leadership grant; resources reject lower numbers |
| **Split brain** | Two nodes both acting as leader (topic 17) |
| **Wall clock** | Time of day (can jump backward/forward with NTP adjustments) |
| **Monotonic clock** | Only moves forward; good for measuring durations, meaningless across machines |
| **Logical clock** | Counters that capture order of events, not real time (Lamport clocks, vector clocks) |

---

## Why it exists

You will run consensus-based systems constantly (etcd, Consul, ZooKeeper, Kafka KRaft controllers, Vault Raft storage, Patroni) and design things that need "only one runs at a time" (schedulers, migrations, Terraform applies, cron leaders). Getting quorum sizes, AZ placement, or locking wrong causes the worst outages: split brain and data corruption.

---

## Analogy

Committee voting by letters (Core). Extras:
- **Lease** = the chairperson's badge expires every 15 minutes unless the committee renews it. A chair who has been locked in a cupboard (paused process) may still *think* they're chair when they get out, but their badge has expired.
- **Fencing token** = each new chair gets a higher badge number; the secretary only accepts instructions with the highest number seen so far.

---

## How it works inside

### Clocks

- Servers sync time with **NTP** (chrony, [Linux 069](../../linux-mastery-rhel/notes/069-time-sync-chrony.md)); typical skew in cloud: sub-millisecond to a few ms (AWS Time Sync can do microseconds), but **can** be seconds when misconfigured, after VM pauses, or during leap seconds.
- **Don't** order events across machines by wall-clock timestamps ("last write wins" with skewed clocks keeps the *wrong* write, topic 19).
- **Don't** measure timeouts with wall clocks (an NTP jump can make a 10 s timeout fire instantly or never). Use monotonic clocks.
- **Do** use: version numbers / sequence numbers from a single authority, database-generated ordering, logical clocks, or (Spanner) bounded-uncertainty clocks (TrueTime) with commit-wait.
- **Process pauses:** a GC (garbage collection) pause, VM live-migration or CPU starvation can freeze a process for seconds. When it wakes, it doesn't know time passed. Any design assuming "I checked my lease 1 ms ago, so I'm still leader" is broken.

### Raft in plain words

```
Roles: Leader | Followers | (Candidate during elections)

1. Election: followers expect heartbeats from the leader. If none arrive within a random
   election timeout (e.g. 150–300 ms, randomized to avoid ties), a follower becomes candidate,
   increments the TERM, votes for itself, asks others for votes.
   Majority votes → it's leader for this term.
2. Log replication: clients send writes to the leader → leader appends to its log →
   sends to followers → once a MAJORITY has stored it, the entry is COMMITTED → applied.
3. Safety: a node only votes for candidates whose log is at least as up to date as its own,
   so a new leader always has every committed entry.
4. Partition: the side with a majority keeps a leader and accepts writes; the minority side
   can't elect a leader (no majority) → refuses writes. No split brain.
```

### Quorum sizes

| Nodes (N) | Quorum | Tolerates failures | Note |
|---|---|---|---|
| 1 | 1 | 0 | Dev only |
| 2 | 2 | 0 | ⚠️ Worse than 1 (two things to fail) |
| 3 | 2 | 1 | Standard |
| 4 | 3 | 1 | No gain over 3 |
| 5 | 3 | 2 | Large/critical clusters; survives 1 maintenance + 1 failure |
| 7 | 4 | 3 | Rarely worth it: every write waits for 4 nodes |

More members = more fault tolerance but **slower writes** (bigger majority to reach) and more replication traffic. etcd recommends 3 or 5.

**Placement:** 3 members across 3 AZs → survives any one AZ. 3 members across 2 AZs (2 + 1) → losing the AZ with 2 loses quorum. For **two-datacenter** setups you need a third site (even a tiny "witness/tie-breaker" node) to keep quorum.

### Leader election and leases (application level)

Many apps need "only one instance does X" (scheduler, outbox relay, controller):

```
Kubernetes Lease object (coordination.k8s.io):
  instance A: create/update Lease "my-scheduler" holder=A, renewTime=now, duration=15s
  instance B: sees holder=A and renewTime recent → stays follower
  A dies → stops renewing → after 15 s B acquires the Lease → becomes leader
```
Kubernetes controllers (kube-controller-manager, kube-scheduler, operators) use exactly this. Libraries: client-go leaderelection, Consul sessions, ZooKeeper ephemeral nodes.

⚠️ A paused old leader may wake up and act before noticing it lost the lease. Hence:

### Fencing tokens

```
Lock service grants: A gets token 33. A pauses (GC) for 20 s. Lease expires.
                     B gets token 34, writes to storage with token 34.
                     A wakes, writes with token 33 → storage rejects (33 < 34) ✅
```
The **resource being protected** must check the token (a DB column with "last token", a conditional write). Without this, a lease-based lock is only "probably exclusive".

### Distributed locks: be careful

- **Redis lock** (`SET lock NX PX 30000`): fine for **efficiency** (avoid two workers doing the same expensive job; a rare duplicate is harmless), not for **correctness** (protecting data from concurrent modification), because of pauses, expiry and failover losing the lock.
- **For correctness:** use a consensus store (etcd, ZooKeeper, Consul) **plus fencing**, or better, avoid locks: use the database's own guarantees (transactions, `SELECT ... FOR UPDATE`, conditional updates, unique constraints) and idempotent operations.
- Terraform state locking (topic 36) is a lock for correctness implemented with **conditional writes** on the state storage (S3 `If-None-Match`): the storage itself enforces exclusivity.

### Failure detection

You can't distinguish "dead" from "slow". Detectors use **timeouts + heartbeats**, trading speed for false positives:
- Short timeouts → fast failover, but false failovers under load (which cause load spikes → more false failovers).
- Long timeouts → slow failover (higher MTTR).
- Use several signals, quorum-based decisions, and backoff on repeated failovers.

---

## Options and trade-offs

| Need | Use | Avoid |
|---|---|---|
| Strongly consistent config/coordination | etcd, Consul, ZooKeeper (3 or 5 nodes over 3 AZs) | DIY consensus |
| "Only one runs" for efficiency | K8s Lease, Redis lock with TTL, DB advisory lock | Assuming it's perfectly exclusive |
| "Only one writes" for correctness | DB transactions / conditional writes / fencing tokens | Plain Redis locks |
| Ordering events | Sequence from one authority, Kafka partition offsets, version columns | Wall-clock timestamps from different hosts |
| Measuring durations | Monotonic clocks | Wall clock differences |
| Unique IDs | UUIDv4/v7, Snowflake IDs (timestamp + worker + sequence) | Timestamps alone |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **etcd quorum lost** | K8s API unavailable/read-only; no changes possible | etcd `has_leader`, API errors | 3/5 members across 3 AZs; restore from snapshot ([K8s 77](../../CKA-Certification-Course-2025/notes/77-etcd-backup-restore-and-dr.md)) |
| **etcd slow disk** | Leader elections, API latency | `etcd_disk_wal_fsync_duration_seconds` p99 > ~10 ms, frequent leader changes | Fast dedicated SSDs, no noisy neighbours |
| **Clock skew** | TLS "not yet valid", wrong LWW winners, token expiry errors | NTP offset metrics | chrony everywhere, alerts on offset |
| **Split brain** | Two leaders, diverging data | Two writers detected | Quorum + fencing; never 2-node clusters for leadership |
| **Paused leader acts after losing lease** | Rare data corruption | Hard to detect | Fencing tokens checked by the resource |
| **Lock expired mid-job** | Duplicate processing | Duplicate effects | Renew leases during work; idempotent jobs |
| **Flapping leader election** | Repeated failovers, instability | Leader-change rate | Tune timeouts, fix resource starvation (CPU throttling of the leader!) |
| **Even number of members** | Lower tolerance than expected; quorum lost in 2-AZ split | Design review | Odd sizes, 3 failure domains |

**3 AM:** "Kubernetes API is timing out, `kubectl` hangs." Check etcd: are members healthy, is there a leader, is fsync latency high (disk)? On managed K8s (EKS/GKE/AKS) the provider runs etcd; check provider status and API server metrics, and look for clients overloading the API (a controller in a hot loop listing everything).

---

## Where you'll use it

- **Kubernetes:** etcd (Raft), leader election for controllers, Lease objects, eventual reconciliation ([K8s 07 · control plane](../../CKA-Certification-Course-2025/notes/07-control-plane-components.md), [K8s 78 · HA control plane](../../CKA-Certification-Course-2025/notes/78-ha-control-plane-and-managed-kubernetes.md)).
- **Databases:** Patroni (etcd/Consul leader key), Aurora storage quorum (topic 17).
- **Vault** integrated Raft storage (topic 54); **Kafka KRaft** controllers; **Consul**.
- **CI/CD:** "one deploy at a time" (GitHub Actions `concurrency`, Jenkins `lock`), which is leader election for pipelines ([GHA 29](../../GitHub-Actions-Basics-To-Production/notes/29-concurrency.md), [Jenkins 38](../../Jenkins/notes/38-when-input-milestone-lock.md)).
- **Terraform state locking** ([Terraform 36](../../Terraform/notes/36-state-locking.md)).

---

## Design exercise

**Prompt:** You run 3 replicas of a "billing-runner" service on Kubernetes. Every hour exactly one replica must generate invoices for all customers. Generating twice would double-bill. Design it.

<details>
<summary><b>Model answer</b></summary>

1. **Leader election** with a Kubernetes **Lease** (client-go leaderelection; lease duration 15 s, renew every 5 s). Only the leader runs the hourly loop. This handles "one at a time" for efficiency.
2. **Correctness doesn't rely on the lease** (a paused ex-leader could still run):
   - Each run is identified by its period: `run_id = billing:2026-11-20T10:00`.
   - Start of run: `INSERT INTO billing_runs(run_id, status) VALUES (...)` with a **unique constraint**. If it fails, another run already claimed this period → stop.
   - Per customer: `INSERT INTO invoices(customer_id, period, ...)` with a **unique constraint on (customer_id, period)** → a duplicate attempt is rejected, never double-billed.
   - Charging the card uses an **idempotency key** = `invoice_id` with the payment provider.
3. **Crash mid-run:** the next leader sees `billing_runs.status = RUNNING` with a stale heartbeat, resumes; already-created invoices are skipped by the unique constraint (idempotent resume).
4. **Monitoring:** alert if a period has no completed run within 30 min; alert on lease flapping.
5. Alternative: a Kubernetes CronJob with `concurrencyPolicy: Forbid` (Kubernetes ensures no overlapping Jobs from that CronJob), but the same DB-level idempotency is still needed (CronJobs can occasionally run twice or be missed).
</details>

---

## Quick recap

- Networks fail, clocks drift, and "dead or slow?" is unknowable: design for all three.
- Consensus (Raft) gives one leader and one ordered log while a majority is connected; minority sides refuse writes.
- Quorum = ⌊N/2⌋+1 → use 3 or 5 members across 3 failure domains; never even numbers for quorum.
- Leases + fencing tokens for leadership; Redis locks only for efficiency; correctness from DB constraints/conditional writes.
- Order events with sequence numbers or versions, not wall-clock timestamps.

### Interview questions

1. **Q: Why does etcd run with 3 or 5 members, not 4?**
   A: Quorum is a majority. 4 members need 3 for quorum, so they tolerate only 1 failure, the same as 3 members, while adding cost and making writes slower. 5 members tolerate 2 failures. Members should span 3 AZs so one AZ loss keeps a majority.

2. **Q: Is a Redis `SET NX PX` lock safe to protect a bank balance update?**
   A: No. The lock can expire while the holder is paused (GC, VM pause), a failover can lose it, and the holder can then write after someone else took the lock. For correctness, use the database's own concurrency control (transactions, conditional updates, unique constraints) or a consensus lock with fencing tokens checked by the resource. Redis locks are fine to avoid duplicate *work* when a rare duplicate is harmless.

3. **Q: Why shouldn't you order events by timestamp?**
   A: Clocks on different machines differ and can jump; a later event can carry an earlier timestamp. Last-write-wins on timestamps then silently keeps the wrong value. Use a single sequencer, partition offsets, version numbers with conditional writes, or logical/vector clocks.
