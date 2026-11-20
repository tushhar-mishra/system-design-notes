# 18 · Partitioning and Sharding

> **Level:** ★★★ advanced
>
> **You will learn:** why you split data across machines (**sharding**) when one primary can't keep up, choosing a **shard key**, **range vs hash vs directory** partitioning, **consistent hashing** step by step, **hot partitions** and how to cool them, **cross-shard queries and transactions**, **resharding** without downtime, alternatives (**distributed SQL**, managed NoSQL), and the operational cost of running shards.
>
> **You need:** topics 16–17.
>
> **Part 3 · Caching and data** · Previous: [17 · Replication](17-replication-and-failover.md) · Next: [19 · Consistency, CAP and PACELC](19-consistency-cap-pacelc.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Replication copies **all** data to more machines (scales reads). **Partitioning/sharding** splits data so **each machine holds a part** (scales writes and storage).

```
               ┌─▶ Shard 0: users with hash(user_id) % 4 == 0
App + router ──┼─▶ Shard 1: ... == 1        (each shard = primary + replicas)
               ├─▶ Shard 2: ... == 2
               └─▶ Shard 3: ... == 3
```

**Picture it.** A library too big for one building. You split it: A–F in building 1, G–M in building 2... (range). Or by a formula on the book ID (hash). Each building has its own staff, so four buildings can serve four times the visitors. But a question like "all books published in 2020" now means visiting every building.

**The one thing to know.** The **shard key** decides everything. Good key: spreads load evenly **and** keeps data that's queried together on the same shard (e.g. `tenant_id` for a SaaS). Bad key: hot spots and cross-shard queries everywhere. Changing it later is very expensive.

**Try it now:** for an app you know, write its top 3 queries. Which single column appears in all of them? That's your shard key candidate.

**You've got it if you can:** choose a shard key with reasons, explain consistent hashing, and describe how to fix a hot partition.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **Partition / shard:** a subset of the data stored on its own node (or group of nodes). "Partition" is the general term; "shard" usually means a partition living on a separate database server.
- **Shard key / partition key:** the column(s) used to decide which shard a row belongs to.
- **Router:** the component that maps a key → shard (in the app, a proxy like Vitess/ProxySQL/Citus coordinator, or built into the database).
- **Horizontal partitioning** (sharding) = split rows. **Vertical partitioning** = split columns/tables (e.g. move the `user_photos` table to another DB). **Functional partitioning** = one DB per service/domain (topic 13).
- **Table partitioning inside one DB** (Postgres declarative partitioning by month) = same idea on one server: helps maintenance (drop old partitions instantly) and query pruning, but not write scale beyond the server.

---

## Why it exists

A single primary has limits: write throughput (CPU, WAL I/O), storage (tens of TB), memory for the working set, and backup/restore time (restoring 50 TB takes many hours → bad RTO). When the estimate (topic 03) says writes or data exceed one big node with headroom, you partition.

**But try these first** (much cheaper): fix queries/indexes, vertical scale, caching (reads), read replicas (reads), moving blobs to object storage, archiving old data, splitting by service (functional partitioning), queues to smooth write spikes. Sharding is a one-way door with big operational cost.

---

## Analogy

The multi-building library (Core). Extras:
- **Hot partition** = one building holds all the bestsellers; the queue is out the door while other buildings are empty.
- **Resharding** = moving books between buildings while the library stays open.
- **Directory** = a front desk with a list "this book is in building 3".

---

## How it works inside

### Partitioning strategies

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Range** | Key ranges: A–F, G–M; or dates | Range scans are efficient (all of March on one shard) | Hot spots (all new writes go to "today" shard; popular ranges) |
| **Hash** | `hash(key) % N` | Even spread | Range queries hit all shards; changing N reshuffles almost everything |
| **Consistent hashing** | Keys and nodes on a hash ring | Adding/removing a node moves only ~1/N of keys | Needs virtual nodes for balance |
| **Directory / lookup** | A table maps key → shard | Flexible: move single tenants, place big tenants alone | The directory is a critical dependency (cache it, replicate it) |
| **Geo** | By region/country | Data residency, locality | Uneven sizes |

### Consistent hashing step by step

```
Hash space as a ring 0 … 2^32-1

         node A (pos 10)
     ●─────────────────●  node B (pos 40)
     │                 │
key k1 (pos 25) → walk clockwise → first node = B
     │                 │
     ●─────────────────●
  node D (pos 90)    node C (pos 70)

Add node E at pos 55: only keys between 40 and 55 move (from C to E). Everything else stays.
Plain hash % N: going from 4 to 5 nodes moves ~80% of keys.
```

**Virtual nodes:** each physical node takes many positions on the ring (e.g. 100–256). This spreads load evenly and, when a node leaves, its keys spread across all others instead of dumping on one neighbour. Used by Cassandra, DynamoDB (internally), memcached client libraries, Envoy's ring-hash LB (topic 09).

### Choosing a shard key

Ask:
1. **Cardinality:** many distinct values? (`country` with 50 values can't spread over 100 shards.)
2. **Even load:** are some values much hotter? (A celebrity user, a giant tenant.)
3. **Query locality:** do most queries include this key? (Then they hit one shard.)
4. **Transactions:** do multi-row transactions stay within one key? (`tenant_id` keeps a tenant's orders + invoices together.)
5. **Growth:** monotonic keys (timestamps, auto-increment IDs) with range partitioning → all writes hit the last shard.

| System | Good shard key | Why |
|---|---|---|
| B2B SaaS | `tenant_id` | Queries and transactions are per tenant; easy to isolate/move big tenants |
| Consumer social app | `user_id` | Most reads/writes are per user |
| IoT metrics | `(device_id, day)` | Spreads devices; bounds partition size by day |
| Chat | `conversation_id` | Messages of one conversation together, ordered |
| Orders (marketplace) | `customer_id` or `order_id` | Depends: customer history queries vs order lookups (you may need a secondary index) |

### Cross-shard operations

- **Queries without the shard key** → **scatter-gather**: ask all shards, merge results. Latency = slowest shard; load × N. Mitigate with a **global secondary index** (a separate table keyed by the other column), or a search index/warehouse fed by CDC.
- **Cross-shard transactions** → two-phase commit (slow, blocking on coordinator failure) or **sagas** (topic 22). Better: choose a key that keeps transactions on one shard.
- **Unique constraints across shards** (unique email) → a separate lookup table/service.
- **IDs:** auto-increment per shard collides. Use UUIDs (v7 is time-ordered, index-friendly), or Snowflake-style IDs (timestamp + shard/worker ID + sequence).

### Hot partitions

| Cause | Fix |
|---|---|
| One huge tenant | Move it to a dedicated shard (directory-based) |
| One hot key (celebrity) | Split key with suffix (`post123#0..#9`) for writes and merge on read; cache reads |
| Time-based range keys | Prefix with hash/bucket (`bucket = hash(id) % 16`, then time) |
| Uneven hash | More virtual nodes / more partitions |

**DynamoDB** specifics (common in interviews): each partition supports up to ~3,000 read units and ~1,000 write units per second; adaptive capacity helps, but a single key above these limits throttles. Design keys with high cardinality.

### Resharding (adding capacity)

```
1. Create new shards (empty)
2. Copy data for moving key ranges (snapshot + catch up via replication/CDC)
3. Dual-write or catch up until in sync
4. Flip routing for those ranges (directory update), briefly pausing writes for those keys
5. Verify, then delete old copies
```

Tools: **Vitess** (MySQL sharding, used by YouTube/Slack/GitHub-scale systems), **Citus** (Postgres), MongoDB balancer, Cassandra adding nodes (streams ranges automatically). **Over-partition** up front (e.g. 1,024 logical partitions mapped onto 8 physical shards) so growth = moving logical partitions, not re-hashing.

---

## Options and trade-offs

| Approach | Pros | Cons | Choose when |
|---|---|---|---|
| Don't shard (vertical + replicas + cache + archive) | Simplest | Ceiling | Until estimates say otherwise (most systems) |
| App-level sharding on Postgres/MySQL | Control, familiar DBs | You own routing, resharding, cross-shard logic | Clear tenant key, strong team |
| Sharding middleware (Vitess, Citus) | Automates routing/resharding | New complex system | Large MySQL/Postgres at scale |
| Distributed SQL (Spanner, CockroachDB, YugabyteDB, Aurora Limitless) | SQL + transactions + automatic sharding | Higher latency per transaction, cost, different failure modes | Need SQL semantics at write scale |
| Managed NoSQL (DynamoDB, Cosmos DB, Bigtable) | Automatic partitioning, no servers | Must design for access patterns; limited queries | Key-based access at scale |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Hot shard** | One shard at 100% CPU, others idle; throttling | Per-shard metrics | Better key, split hot keys, move tenant |
| **Scatter-gather overload** | Queries without shard key slow everything | Query patterns per shard | Secondary index / search index / warehouse |
| **One shard down** | Only users on that shard affected (partial outage) | Per-shard health | Each shard has replicas + failover; this isolation is a benefit |
| **Resharding bug** | Missing/duplicated rows | Row-count checks, checksums | Verify, dual-read comparison, reversible steps |
| **Directory service down** | All routing fails | Router errors | Cache mappings in apps; replicate directory |
| **Unbounded partition growth** | One partition too large to move/compact | Partition size metrics | Composite keys with time bucket |
| **Schema migrations ×N** | Some shards migrated, some not | Migration tracking per shard | Tooling that applies and tracks per shard; backward-compatible migrations (topic 35) |

**3 AM:** "Shard 7 is at 100% CPU, tenant X's dashboards time out." Look at top queries on shard 7: usually one tenant's batch job or a hot key. Mitigate: rate-limit that tenant (topic 25), add read replicas for shard 7, kill the runaway job. Long-term: isolate big tenants on dedicated shards.

---

## Where you'll use it

- **Kafka partitions** are the same idea: the key → partition, ordering per partition, parallelism = partitions (topic 21).
- **Prometheus/Thanos/Mimir** shard series by hash across ingesters (topic 50).
- **Log platforms** shard indexes by time and tenant (topic 49).
- **Kubernetes** at very large scale: shard clusters by team/region (topic 38). etcd itself is not sharded, one reason clusters have size limits.
- **Terraform state** "sharding": one state per component/environment to limit blast radius (topic 36): same principle.

---

## Design exercise

**Prompt:** A multi-tenant SaaS on one Postgres: 4,000 tenants, 8 TB, 12,000 writes/s at peak and growing 2×/year. The top 5 tenants generate 40% of load. Design the partitioning.

<details>
<summary><b>Model answer</b></summary>

- **Key:** `tenant_id`. All app queries are tenant-scoped; transactions stay within a tenant.
- **Routing:** **directory-based**: a small, replicated `tenant → shard` table (cached in every app instance, refreshed on change). Lets us place tenants individually.
- **Layout:** start with 8 shards (each Aurora/RDS Postgres Multi-AZ, sized for ~2,500 writes/s with headroom). The **top 5 tenants get dedicated shards** (or two share one) so their load doesn't affect others; the other 3,995 are spread by hash across the remaining shards.
- **Moving tenants:** logical replication/CDC of a tenant's rows to the target shard, catch up, brief write pause for that tenant, flip directory entry, verify counts, delete old rows. Practice this; it's also how we rebalance.
- **Cross-tenant needs:** admin/analytics queries go to a warehouse fed by CDC from all shards, not scatter-gather on production.
- **IDs:** UUIDv7 to avoid collisions and keep index locality.
- **Ops:** migrations run per shard with a tracking table; dashboards per shard; per-tenant rate limits.
- Alternative: Citus (distributes by `tenant_id` automatically, co-locates tables) to avoid building the router. Trade-off: new technology to operate vs custom routing code.
</details>

---

## Quick recap

- Replication scales reads; partitioning scales writes and storage. Exhaust simpler options first.
- Shard key: high cardinality, even load, query locality, transactions on one shard.
- Range (good scans, hot spots), hash (even, no ranges), consistent hashing (minimal movement), directory (flexible placement).
- Avoid cross-shard queries/transactions by design; use global indexes, CDC to warehouses, sagas.
- Over-partition logically; plan resharding and hot-tenant isolation from day one.

### Interview questions

1. **Q: How do you pick a shard key?**
   A: From the access patterns: the key should appear in nearly all queries, keep related data and transactions together, have high cardinality, and spread load evenly. I check for hot values (big tenants, celebrities) and monotonic growth. For a B2B SaaS that's usually `tenant_id` with directory-based placement so big tenants can be isolated.

2. **Q: Why consistent hashing instead of `hash % N`?**
   A: With modulo, changing N remaps almost every key, causing massive data movement or cache misses. Consistent hashing places nodes and keys on a ring; adding or removing a node moves only the keys in its arc (~1/N). Virtual nodes keep the distribution even and spread a failed node's load across many others.

3. **Q: One DynamoDB partition key gets throttled. How do you fix it?**
   A: That key exceeds per-partition throughput. Options: write sharding (append a random or computed suffix and aggregate on read), cache hot reads (DAX or app cache), redesign the key for higher cardinality, or buffer writes through a queue to smooth bursts. On-demand capacity doesn't fix a single hot key.
