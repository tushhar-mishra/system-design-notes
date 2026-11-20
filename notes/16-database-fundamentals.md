# 16 · Database Fundamentals

> **Level:** ★★ intermediate
>
> **You will learn:** the database families (**relational/SQL, key-value, document, wide-column, graph, time-series, search**), **ACID** transactions in plain words, **isolation levels**, **indexes** (B-tree vs LSM-tree) and why a missing index is the #1 production performance bug, **query cost** and `EXPLAIN`, **connection management**, **locks** and long transactions, **managed vs self-hosted** databases, and a **decision guide** for choosing a database in an interview.
>
> **You need:** topics 03, 04, 11, 14.
>
> **Part 3 · Caching and data** · Previous: [15 · CDN](15-cdn-and-edge.md) · Next: [17 · Replication and failover](17-replication-and-failover.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The database is the **source of truth** and almost always the hardest part to scale and protect. Choose by **data shape** and **access pattern**, not hype.

```
Relational (Postgres, MySQL)  → structured data, relations, transactions. DEFAULT choice.
Key-value (DynamoDB, Redis)   → get/put by key at massive scale, simple access patterns
Document (MongoDB)            → flexible JSON documents, read whole objects
Wide-column (Cassandra)       → huge write volume, time-ordered data per key
Time-series (Prometheus, TimescaleDB) → metrics, sensor data
Search (OpenSearch/Elasticsearch)     → full-text search, log search
Graph (Neo4j)                 → relationship-heavy queries
```

**Picture it.** A library. Relational = a catalogue with cross-references (author → books → loans), strict rules. Key-value = a coat check: give ticket, get coat, nothing else. Document = each book comes with its own folder of everything about it. Search = the index at the back of every book.

**The one thing to know.** **Indexes** make reads fast (find a row among millions in a few page reads instead of scanning everything). Missing indexes cause most "the database is slow" incidents. Every index slows writes a bit and uses space.

**Try it now (Postgres):** `EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;` → `Seq Scan` on a big table means no usable index.

**You've got it if you can:** explain ACID, pick a database family for 4 different workloads, and explain why a query became slow as a table grew.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Families

| Family | Data model | Strengths | Weaknesses | Examples (AWS / GCP / Azure / OSS) |
|---|---|---|---|---|
| **Relational (SQL)** | Tables, rows, joins, schema | Transactions, flexible queries, constraints, mature tooling | Write scaling beyond one primary is hard (sharding, topic 18) | RDS/Aurora · Cloud SQL/AlloyDB/Spanner · Azure SQL/Database for PostgreSQL · PostgreSQL, MySQL |
| **Key-value** | key → value | Massive scale, predictable ms latency | Only access by key (or designed indexes) | DynamoDB · Bigtable/Firestore · Cosmos DB · Redis |
| **Document** | JSON-like documents | Flexible schema, read whole aggregate in one go | Joins and cross-document transactions weaker | DocumentDB · Firestore · Cosmos DB · MongoDB |
| **Wide-column** | Partition key → sorted rows | Very high write throughput, linear scale | Query only by designed keys; eventual consistency | Keyspaces · Bigtable · Cosmos DB (Cassandra API) · Cassandra, ScyllaDB |
| **Time-series** | (series, timestamp) → value | Compression, downsampling, time queries | Narrow use | Timestream · Cloud Monitoring/Bigtable · Azure Data Explorer · Prometheus, TimescaleDB, InfluxDB |
| **Search** | Inverted index of documents | Full-text, aggregations | Not a source of truth; heavy to run | OpenSearch Service · (Elastic on GCP) · Azure AI Search · Elasticsearch, OpenSearch |
| **Graph** | Nodes and edges | Multi-hop relationship queries | Niche, scaling | Neptune · (Spanner Graph) · Cosmos DB Gremlin · Neo4j |
| **Data warehouse (OLAP)** | Columnar tables | Huge analytical scans | Not for transactions | Redshift · BigQuery · Synapse/Fabric · ClickHouse, Snowflake |

> **OLTP** (Online Transaction Processing) = many small reads/writes (your app's DB). **OLAP** (Online Analytical Processing) = few huge analytical queries (reports). Don't run heavy analytics on the OLTP primary; replicate to a warehouse or a replica.

### ACID

A **transaction** is a group of operations that succeed or fail together. ACID describes the guarantees:

- **A – Atomicity:** all or nothing. Debit and credit both happen, or neither does.
- **C – Consistency:** the DB moves from one valid state to another (constraints like "balance ≥ 0", foreign keys hold).
- **I – Isolation:** concurrent transactions don't see each other's half-done work (how strictly depends on the **isolation level**).
- **D – Durability:** once committed, it survives a crash (written to the **WAL**, Write-Ahead Log, on disk before the commit returns).

**Isolation levels** (weakest → strongest): Read Uncommitted → **Read Committed** (Postgres default) → **Repeatable Read** (MySQL InnoDB default) → **Serializable**. Stronger = fewer anomalies (like two transactions both reading "1 seat left" and both booking it), but more locking/aborts and less throughput.

**BASE** (Basically Available, Soft state, Eventually consistent) is the informal label for many NoSQL systems that relax consistency for availability and scale (topic 19).

---

## Why it exists

Different data and access patterns need different storage engines. One size doesn't fit all, but **one relational database fits far more than people think**: a single well-tuned Postgres on a large instance handles thousands of transactions per second and terabytes of data. Start there unless a requirement clearly points elsewhere.

---

## Analogy

The library (Core). **Index** = the library catalogue: without it, find a book by walking every shelf (full table scan). With it, look up the title, go to shelf 42. But every new book must also be added to every catalogue (indexes slow writes).

---

## How it works inside

### How a query is executed

```
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
  1. Parse → plan: the query planner estimates costs using table statistics
  2. Chooses: Index Scan on idx_orders_customer_created  (vs Seq Scan of 50 M rows)
  3. Reads index pages (B-tree: ~3–4 levels deep even for millions of rows)
  4. Fetches 20 matching rows from table pages (from RAM buffer pool if cached, else disk)
  5. Returns rows
```

### Indexes

- **B-tree** (most relational DBs): a balanced sorted tree. Finds a key in O(log n) page reads. Supports equality and range (`>`, `BETWEEN`, `ORDER BY`).
- **Composite index** `(customer_id, created_at)` serves `WHERE customer_id = ? ORDER BY created_at`. Column order matters: the index is useful for the leftmost columns.
- **LSM-tree** (Log-Structured Merge tree: Cassandra, RocksDB, many NoSQL): writes go to memory then are flushed to sorted files and merged in the background (**compaction**). Very fast writes; reads may check several files; compaction uses I/O.
- **Cost of indexes:** each insert/update must update every index; more storage; too many indexes slow writes.

**Why queries get slow as tables grow:** a query without a suitable index does a **full table scan**: fine at 10,000 rows (ms), terrible at 50 M rows (seconds), and it reads so many pages it evicts useful data from the buffer cache, slowing *other* queries too.

### Connections

Each Postgres connection is a separate OS process (~5–10 MB RAM). Thousands of connections waste memory and CPU on context switching. → **connection pooling** (topic 04: PgBouncer, RDS Proxy).

### Locks and long transactions

- Writes take **row locks**; schema changes (`ALTER TABLE`) may take **table locks** that block everything (topic 35).
- A long-running transaction (an analyst's query left open, a batch job) holds locks and, in Postgres, prevents **vacuum** from cleaning up old row versions → table **bloat** and slowdowns.
- **Deadlock** = two transactions each wait for the other's lock; the DB kills one.

### Managed vs self-hosted

| | Managed (RDS/Aurora, Cloud SQL, Azure Database) | Self-hosted (VMs or K8s operators) |
|---|---|---|
| Backups, PITR, patching, failover | Built in | You build and test it |
| Control (extensions, versions, tuning) | Limited | Full |
| Cost | Higher per hour, lower people cost | Lower infra, high people cost |
| On Kubernetes | — | Possible with operators (CloudNativePG, Zalando, Percona) ([K8s 67 · operators](../../CKA-Certification-Course-2025/notes/67-operators.md)) — needs strong storage and DR skills |
| Choose | **Default** for most teams | Special needs, cost at very large scale, on-prem |

---

## Options and trade-offs: a decision guide

```
Do you need transactions across multiple records/tables, joins, ad-hoc queries?
   └─ yes → Relational (Postgres/MySQL). Scale reads with replicas + cache.
            Writes exceed one big primary? → shard (18) or distributed SQL (Aurora Limitless, Spanner, CockroachDB, YugabyteDB)
   └─ no → Is access always by a known key at huge scale with predictable latency?
            └─ yes → Key-value / wide-column (DynamoDB, Cassandra). Design keys from access patterns.
Full-text search or log search? → Search engine (fed from the source of truth)
Metrics? → Time-series DB
Files/blobs? → Object storage, NOT a database (topic 20)
Analytics/reporting? → Warehouse, fed by CDC/ETL
```

| Workload | Pick | Reason |
|---|---|---|
| E-commerce orders, payments | Postgres/MySQL (Aurora) | Transactions, constraints, flexible queries |
| Session store, feature flags | Redis / DynamoDB | Key lookups, low latency |
| IoT events, 1 M writes/s | Cassandra / DynamoDB / time-series DB | Write throughput, partitioned by device |
| Product search | OpenSearch fed from Postgres | Full-text relevance |
| Terraform state locks, CI metadata | DynamoDB / Postgres | Simple, durable |
| Logs | Object storage + index (Loki) or OpenSearch | Volume, search (topic 49) |

**Polyglot persistence** = using several databases, each for its job. Trade-off: each one is another system to run, secure, back up and keep in sync.

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Missing index / bad plan** | One endpoint slow, DB CPU high | Slow query log, `pg_stat_statements`, Performance Insights | Add index (concurrently!), fix query; review query plans in CI for big tables |
| **Connection exhaustion** | `too many connections` | Connection count | Pooling, smaller pools (topic 04) |
| **Lock contention / blocking DDL** | Requests hang, pile up | `pg_locks`, blocked queries | Short transactions, `lock_timeout`, online schema change (topic 35) |
| **Long transaction / vacuum blocked** | Bloat, slowdowns, eventually transaction ID wraparound risk in Postgres | Oldest transaction age, bloat metrics | Kill idle-in-transaction sessions (`idle_in_transaction_session_timeout`), monitor |
| **Disk full** | Writes fail, DB may stop | Free storage alert at 80% | Storage autoscaling, retention, alerts |
| **Replica used for analytics slows primary** (via replication conflicts) or analytics on primary | Production latency up during reports | Query source | Dedicated analytics replica/warehouse |
| **N+1 queries** (ORM loads a list, then 1 query per item) | 101 queries per page | Query count per request in traces | Eager loading, batch queries |
| **Hot rows** | Lock waits on one row (global counter) | Lock wait stats | Shard counters, batch updates |

**3 AM:** "DB CPU 100%." Open the top queries by total time (`pg_stat_statements` / Performance Insights). Usually one query: a new deploy introduced it, or a plan changed after statistics updated. Mitigate: roll back the deploy, kill runaway queries, add the index concurrently. Long-term: query review and slow-query alerts.

---

## Where you'll use it

- **Every design** has a data layer: always say which DB, why, how it scales, how it's backed up.
- **Platform tools are DB-backed:** Jenkins (files), SonarQube (Postgres), Grafana, Argo CD (K8s API/etcd), Vault (storage backend).
- **Terraform:** `aws_db_instance` / `aws_rds_cluster` with Multi-AZ, backups, parameter groups.
- **Kubernetes:** StatefulSets and operators for self-hosted DBs ([K8s 40](../../CKA-Certification-Course-2025/notes/40-statefulsets.md)).

---

## Design exercise

**Prompt:** Choose the data stores for an internal developer platform: (a) service catalog (services, owners, dependencies), (b) build logs (5 TB/month), (c) deployment events for dashboards ("deploys per day per team"), (d) search across service docs.

<details>
<summary><b>Model answer</b></summary>

- **(a) Service catalog:** **Postgres** (managed, Multi-AZ). Relations (team → services → dependencies), constraints, modest size. Dependency graph queries can be recursive CTEs (Common Table Expressions); a graph DB isn't justified.
- **(b) Build logs:** **object storage** (S3) with lifecycle rules (Standard → infrequent access after 30 days → delete/archive after 1 year). Metadata (build ID → S3 key) in Postgres. Not a DB: 5 TB/month of blobs.
- **(c) Deployment events:** append to Postgres (low volume: maybe thousands/day) with an index on `(team_id, created_at)`; for dashboards, either query directly or push to the metrics system as counters. If it grows to analytics scale, CDC into a warehouse.
- **(d) Doc search:** **OpenSearch** (managed) indexed from the docs source (Git) by a pipeline; it's a derived index, rebuildable, not a source of truth.

Trade-off: two data stores beyond object storage (Postgres + OpenSearch); we accept OpenSearch only because full-text relevance is a real requirement. Postgres full-text search could be a simpler first step.
</details>

---

## Quick recap

- Default to a managed relational DB; pick other families for specific shapes (key-value scale, search, time-series, blobs → object storage).
- ACID: atomic, consistent, isolated, durable; isolation levels trade anomalies for throughput.
- Indexes (B-tree/LSM) make reads fast and writes slower; missing indexes cause most DB incidents.
- Pool connections, keep transactions short, beware blocking DDL.
- Managed DBs give backups, PITR, failover and patching; self-hosting means you own all of it.

### Interview questions

1. **Q: SQL or NoSQL for a new service?**
   A: Start from access patterns and consistency needs. If there are relations, transactions and ad-hoc queries, I choose relational (Postgres), which scales further than people expect with replicas and caching. I choose key-value/wide-column when access is always by key at a scale or latency one primary can't meet, and I've designed the keys around the queries.

2. **Q: A query that took 5 ms last month takes 3 s today. What happened?**
   A: Most often table growth with a missing or unused index (a full scan that was cheap is now huge), a plan change after statistics updates, or lock contention/bloat. I'd run `EXPLAIN ANALYZE`, compare plans, check `pg_stat_statements`, and look for blocking locks or long transactions.

3. **Q: What does "durable" mean, and how does the DB guarantee it?**
   A: A committed transaction survives crashes. The DB writes changes to the write-ahead log and flushes it to disk (fsync) before acknowledging the commit; on restart it replays the log. Durability across machine loss additionally needs replication and backups (topics 17, 41).
