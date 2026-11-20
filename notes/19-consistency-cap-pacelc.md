# 19 · Consistency, CAP and PACELC

> **Level:** ★★★ advanced
>
> **You will learn:** what **consistency** means in distributed systems (and how it differs from the C in ACID), the **consistency models** from strong (**linearizable**) to **eventual**, the session guarantees (**read-your-writes**, monotonic reads), **quorums** (`R + W > N`), what the **CAP theorem** really says (and the common misreading), **PACELC** (the latency trade-off when nothing is broken), and how real systems (Postgres, DynamoDB, Cassandra, etcd, S3, Spanner) sit on this map.
>
> **You need:** topics 16–18.
>
> **Part 3 · Caching and data** · Previous: [18 · Partitioning](18-partitioning-and-sharding.md) · Next: [20 · Storage](20-storage-block-file-object.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** When data has several copies, a reader might see an old copy. **Consistency models** describe what readers are allowed to see.

```
Strong (linearizable): every read sees the latest committed write. Like one single copy.
Eventual:              copies converge "soon"; reads may be stale meanwhile.
In between:            read-your-writes, monotonic reads, causal, bounded staleness...
```

**CAP:** during a **network Partition**, a system must choose: stay **Consistent** (refuse some requests) or stay **Available** (answer, possibly with stale/conflicting data). **PACELC** adds: **Else** (no partition), choose between **Latency** and **Consistency**.

**Picture it.** Two bank branches sharing one ledger by phone. The phone line drops (partition). Either both branches stop serving withdrawals until the line is back (consistent, not available) or they keep serving and risk the same money being withdrawn twice (available, not consistent). Even when the line works, calling the other branch before every withdrawal makes customers wait (latency vs consistency).

**The one thing to know.** Choose consistency **per operation**, not per system: money movements, inventory reservation and locks need strong consistency; likes, view counts, feeds and search can be eventual.

**Try it now:** for an app you know, list 5 operations and label each "must be strong" or "eventual OK". That list is your consistency design.

**You've got it if you can:** state CAP correctly, compute a quorum, and choose consistency per feature with reasons.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**Two different "C"s:**
- **ACID consistency** = the database keeps its rules (constraints) (topic 16).
- **Distributed consistency** = which **version** of data a reader sees when there are multiple copies (this topic).

**Consistency models (strongest → weakest):**

| Model | Guarantee in plain words |
|---|---|
| **Linearizable (strong)** | Behaves like one copy; once a write is acknowledged, every later read (by anyone) sees it. |
| **Sequential** | Everyone sees operations in the same order (but maybe a bit late). |
| **Causal** | If A caused B (I read your post, then reply), everyone sees A before B. Unrelated writes may appear in different orders. |
| **Read-your-writes** (session) | *You* always see your own writes. |
| **Monotonic reads** (session) | You never see data go "back in time". |
| **Bounded staleness** | Reads are at most X seconds / N versions old. |
| **Eventual** | If writes stop, all copies converge eventually. No timing promise. |

**Quorum:** with **N** replicas, a write must be confirmed by **W** replicas and a read must ask **R** replicas. If **R + W > N**, every read set overlaps every write set → the read sees at least one copy with the latest write.

---

## Why it exists

Copies exist for availability and scale (topics 17–18). Copies + network delays/failures = possible disagreement. You must decide, consciously, what each feature tolerates. Getting it wrong means double-sold tickets, negative balances, or (in the other direction) needless outages and latency.

---

## Analogy

Two bank branches (Core). Extras:
- **Quorum** = with 3 branch ledgers, a deposit must be recorded in at least 2 (W=2), and a balance check must look at at least 2 (R=2). Any 2 overlap with any other 2, so the balance check always finds the latest deposit.
- **Eventual consistency** = a shared notice board in each branch, updated by a courier every hour.

---

## How it works inside

### CAP, stated correctly

- **Partition tolerance is not optional** in a distributed system: networks *will* partition (switch failures, AZ isolation, misconfigured security groups).
- So the real choice is **during a partition**: **CP** (refuse/err on the minority side to stay consistent) or **AP** (keep answering on both sides, reconcile later).
- When there's **no** partition, a system can be both consistent and available. CAP says nothing about that case → PACELC does.

```
During a partition:
  CP system (etcd, ZooKeeper, Spanner, single-primary SQL with sync standby):
      minority side → errors/timeouts for writes (and strong reads)
  AP system (Cassandra/DynamoDB with eventual reads, DNS, CDN caches):
      both sides keep serving → stale reads, conflicting writes resolved later
```

### PACELC

**If Partition: Availability or Consistency; Else: Latency or Consistency.**

| System | P → A or C | E → L or C | Notes |
|---|---|---|---|
| Single-primary Postgres/MySQL (sync standby) | C (minority can't write) | C (reads on primary) | Async replicas → "EL" for replica reads |
| DynamoDB | A (eventual reads default) | L (eventual read default); C with strongly consistent reads (2× read cost, same-region only) | Global tables: last-writer-wins across regions |
| Cassandra | A | L (tunable with consistency levels) | `QUORUM` gives C-ish behaviour per query |
| etcd / ZooKeeper / Consul | C | C | Raft/ZAB consensus; minority partition refuses writes (topic 26) |
| Spanner / CockroachDB | C | C (pays latency for consensus) | Spanner uses TrueTime (GPS/atomic clocks) for ordering |
| S3 | — | C | Strong read-after-write consistency for all operations since Dec 2020 |
| DNS, CDN | A | L | Eventual by design (TTLs) |

### Quorums worked

N = 3 replicas:

| W | R | R+W>N? | Behaviour |
|---|---|---|---|
| 3 | 1 | yes | Fast reads; a write fails if any replica is down |
| 2 | 2 | yes | **Balanced (QUORUM/QUORUM)**: tolerates 1 replica down for both reads and writes |
| 1 | 3 | yes | Fast writes; reads need all |
| 1 | 1 | **no** | Fastest, may read stale data (eventual) |

Cassandra consistency levels: `ONE`, `QUORUM`, `LOCAL_QUORUM` (quorum within the local data centre: avoids cross-region latency), `ALL`. Even with quorums, edge cases (sloppy quorums, concurrent writes with clock-based last-write-wins) mean quorum ≠ perfect linearizability.

### Conflict resolution (AP / multi-leader systems)

- **Last-writer-wins (LWW):** keep the write with the latest timestamp. Simple; **silently loses** concurrent writes; depends on clocks (topic 26).
- **Version vectors / application merge:** keep both versions, let the app merge (shopping cart: union of items).
- **CRDTs:** counters, sets, maps designed so merges always converge (used in collaborative editing, some Redis Enterprise/Riak features).

### Choosing per operation (a typical e-commerce)

| Operation | Model | How |
|---|---|---|
| Place order / payment | Strong | Primary DB, transaction, idempotency key |
| Reserve last item in stock | Strong | Conditional update on primary |
| User sees their new order in the list | Read-your-writes | Read from primary for that user briefly |
| Product catalogue | Eventual (seconds–minutes) | Cache, CDN, replicas |
| Review counts, likes | Eventual | Async counters |
| Search results | Eventual (seconds) | Index fed by events |
| Distributed lock / leader election | Linearizable | etcd/ZooKeeper/consensus |

---

## Options and trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Strong everywhere | Simple reasoning, no anomalies | Latency (consensus/sync), lower availability during partitions, harder to scale globally |
| Eventual everywhere | Speed, availability, global scale | Anomalies, conflict handling in app code, confused users |
| **Mixed per operation** ✅ | Best of both where it matters | Must document and test which path is which |
| Session guarantees (read-your-writes) | Fixes most user-visible weirdness cheaply | Some routing logic |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Stale read used for a decision** | Oversold stock, double booking | Business reconciliation | Strong read / conditional write at decision points |
| **LWW loses writes** | Updates "disappear" in multi-region | User reports, audit diffs | Avoid concurrent writes to same item across regions (home region per record), merge logic, CRDTs |
| **Clock skew with LWW** | Older write wins | Clock monitoring | NTP, logical clocks, avoid LWW for important data |
| **CP system loses quorum** | Writes fail (e.g. etcd → K8s API read-only/unavailable) | Quorum/leader metrics | Odd number of members across 3 AZs; never 2 AZs for quorum systems |
| **Users confused by eventual consistency** | "My change didn't save" support tickets | Ticket patterns | Read-your-writes, optimistic UI |
| **Assuming S3/DynamoDB semantics wrongly** | Rare data races | Code review | Know each store's guarantees (e.g. DynamoDB strongly consistent reads are opt-in) |

**3 AM:** "etcd lost quorum, the Kubernetes API is down." A CP system did exactly what it's designed for. Running workloads keep running (kubelets keep pods alive), but nothing can change. Restore quorum (bring back members or restore from snapshot, [K8s 77](../../CKA-Certification-Course-2025/notes/77-etcd-backup-restore-and-dr.md)). Prevention: 3 or 5 members spread over 3 AZs.

---

## Where you'll use it

- **Interview vocabulary:** "Which side of CAP?" Answer per operation, mention PACELC latency.
- **Kubernetes:** etcd is CP; controllers are eventually consistent (they reconcile toward desired state) ([K8s 10](../../CKA-Certification-Course-2025/notes/10-what-happens-after-kubectl-apply.md)).
- **Terraform state:** needs a strongly consistent store + locking (S3 strong consistency + lock file, topic 36).
- **GitOps:** cluster state converges eventually to Git (topic 33).
- **Multi-region design** (topic 42): the hardest consistency decisions.

---

## Design exercise

**Prompt:** A global ticket-booking system for concerts. Users browse events worldwide; seats for a popular concert sell out in minutes. Choose the consistency for: browsing events, seat map display, holding a seat for 10 minutes, payment, and the "my tickets" page.

<details>
<summary><b>Model answer</b></summary>

- **Browsing events:** eventual. CDN + caches, replicas in each region. Staleness of a minute is fine.
- **Seat map display:** eventual with short staleness (a few seconds); it's a hint, not a promise. Seats can show "available" and fail on hold; UI handles it.
- **Holding a seat:** **strong (linearizable)**. One authoritative store for each event (its "home region", near the venue), a conditional write: `UPDATE seats SET held_by=?, hold_expires=now()+10min WHERE seat_id=? AND (held_by IS NULL OR hold_expires < now())`. Users far away pay extra latency (one cross-region round trip, ~150–250 ms): acceptable for correctness. (Alternative: a globally consistent DB like Spanner, more cost.)
- **Payment:** strong, idempotency key, in the event's home region; the hold must still be valid at commit (same transaction).
- **"My tickets":** read-your-writes: read from the home region right after purchase (or show the confirmation from the write response), eventual afterwards.
- **During a partition** isolating the home region: holds/payments for that event **fail** (CP choice: better to say "try again" than double-sell); browsing keeps working (AP).
</details>

---

## Quick recap

- Distributed consistency = which version readers see; models range from linearizable to eventual.
- Quorum: R + W > N gives overlapping reads/writes (QUORUM/QUORUM with N=3 tolerates one failure).
- CAP: during a partition, choose consistency or availability. PACELC: otherwise, choose latency or consistency.
- Decide per operation: strong at decision points (money, stock, locks), eventual for browsing/counters/search.
- Session guarantees (read-your-writes) fix most user-visible weirdness cheaply.

### Interview questions

1. **Q: Explain CAP and a common misunderstanding.**
   A: In the presence of a network partition, a distributed system can't be both linearizably consistent and available on every node; it must choose. The misunderstanding is "pick 2 of 3" as if you could drop partition tolerance. You can't; partitions happen. The choice is CP or AP behaviour during partitions, and PACELC adds the everyday latency vs consistency trade-off.

2. **Q: With N=3, why is QUORUM writes + QUORUM reads popular?**
   A: W=2 and R=2 give R+W=4 > 3, so every read overlaps the latest successful write, and both reads and writes still succeed with one replica down. It balances latency, availability and consistency. `LOCAL_QUORUM` keeps that within a region to avoid cross-region latency.

3. **Q: Where would you accept eventual consistency in a payments product?**
   A: Not in moving money or balances used for decisions. But dashboards, transaction history lists (with read-your-writes for the user's latest), analytics, notifications, search, and fraud-score features can be eventual. Each decision point (authorize, capture, refund) reads and writes the strongly consistent source of truth.
