# 21 · Queues, Pub/Sub and Event Streaming

> **Level:** ★★ intermediate
>
> **You will learn:** why systems talk **asynchronously**, the three shapes (**work queue, pub/sub topic, log/stream**), how **SQS, SNS, Kafka** (and GCP Pub/Sub, Azure Service Bus/Event Hubs) differ, **delivery semantics** (at-most-once, at-least-once, "exactly-once"), **acknowledgements and visibility timeouts**, **ordering and partitions**, **consumer groups**, **DLQs** (Dead Letter Queues) and poison messages, **backpressure** and **consumer lag**, **retention and replay**, and how to size a Kafka cluster.
>
> **You need:** topics 03, 06, 12, 18.
>
> **Part 3 · Caching and data** · Previous: [20 · Storage](20-storage-block-file-object.md) · Next: [22 · Async patterns](22-async-patterns.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Instead of calling another service and waiting, a **producer** drops a **message** into a **broker**; **consumers** process it when they can. This decouples them in **time** (consumer can be down), **speed** (spikes are buffered), and **knowledge** (producer doesn't know who consumes).

```
Work queue:   Producer ─▶ [ queue ] ─▶ one of N workers takes each message      (SQS, RabbitMQ)
Pub/Sub:      Producer ─▶ [ topic ] ─▶ every subscriber gets a copy             (SNS, GCP Pub/Sub)
Log/stream:   Producer ─▶ [ partitioned, retained log ] ─▶ consumer groups read at their own offsets (Kafka, Kinesis)
```

**Picture it.** Work queue = the ticket rail in a kitchen: each order is taken by one cook. Pub/sub = a newsletter: everyone subscribed gets their copy. Log = a ledger book everyone can read from any page, at their own pace, and re-read later.

**The one thing to know.** Real systems deliver **at least once**: a message can arrive **twice** (consumer crashed after processing but before acking). So **every consumer must be idempotent** (topic 22).

**Try it now:** `aws sqs get-queue-attributes --queue-url <url> --attribute-names ApproximateNumberOfMessages ApproximateAgeOfOldestMessage`: the two numbers you alert on.

**You've got it if you can:** choose between SQS, SNS+SQS fan-out and Kafka for a case, explain visibility timeout and DLQ, and explain why Kafka consumers can't exceed partitions.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **Message / event** | A small record: "OrderPlaced {id: 9001, …}". Events describe something that happened (past tense). Commands ask for something to be done. |
| **Producer / publisher** | Sends messages |
| **Consumer / subscriber** | Receives and processes messages |
| **Broker** | The system holding messages (SQS, RabbitMQ, Kafka) |
| **Ack** (acknowledgement) | Consumer tells broker "done, delete/advance" |
| **Visibility timeout** (SQS) | After a consumer receives a message, it's hidden from others for N seconds; if not deleted in time, it reappears |
| **DLQ** (Dead Letter Queue) | Where messages go after failing N times, so they don't block or loop forever |
| **Partition** (Kafka) | An ordered, append-only log; a topic has many partitions for parallelism |
| **Offset** | Position of a message in a partition; consumers track their offset |
| **Consumer group** | Consumers sharing the work of a topic; each partition is read by one consumer in the group |
| **Consumer lag** | How many messages a consumer group is behind the newest |
| **Retention** | How long messages are kept (SQS: up to 14 days; Kafka: configurable, e.g. 7 days or by size, or forever with compaction/tiered storage) |

**Products:**

| Shape | AWS | GCP | Azure | Open source |
|---|---|---|---|---|
| Work queue | **SQS** (standard, FIFO) | Pub/Sub (pull subscription), Cloud Tasks | Service Bus queues, Storage Queues | RabbitMQ, Redis Streams |
| Pub/sub fan-out | **SNS** (+ SQS per subscriber), EventBridge | Pub/Sub topics | Service Bus topics, Event Grid | RabbitMQ exchanges, NATS |
| Log / stream | **Kinesis**, **MSK** (managed Kafka) | Pub/Sub (with replay), Managed Kafka | Event Hubs (Kafka API) | **Kafka**, Redpanda, Pulsar |

---

## Why it exists

1. **Absorb spikes:** 10,000 orders arrive in a minute; workers process 1,000/min; the queue holds the rest. No one gets a 503.
2. **Decouple availability:** email service down? Orders still succeed; emails go out when it's back.
3. **Fan-out:** one "OrderPlaced" event feeds billing, shipping, analytics, notifications, without the order service calling each.
4. **Slow work off the request path:** video encoding, PDF generation, image resizing, CI builds.
5. **Replay and audit** (logs): rebuild a search index or a new service's database by re-reading history.

---

## Analogy

Kitchen ticket rail / newsletter / ledger (Core). Extras:
- **Visibility timeout** = a cook takes a ticket; if they don't finish within 10 minutes, the ticket goes back on the rail for someone else.
- **DLQ** = a ticket that every cook has failed 5 times goes to the manager's tray for a human to look at.
- **Partition ordering** = each table's orders are on their own clip, processed in order; different tables in parallel.

---

## How it works inside

### SQS work queue lifecycle

```
Producer ─SendMessage─▶ [SQS]
Worker A ─ReceiveMessage (long poll 20 s)─▶ gets msg M; M becomes INVISIBLE for 60 s (visibility timeout)
   ├─ success → DeleteMessage(M)  ✅ gone
   ├─ crash/timeout → after 60 s M becomes visible again → Worker B receives it (DUPLICATE work possible)
   └─ fails 5 times (maxReceiveCount=5) → M moved to DLQ
```

- **Standard queues:** nearly unlimited throughput, at-least-once, best-effort ordering.
- **FIFO queues:** ordered within a **message group ID**, deduplication within 5 minutes, lower throughput (high-throughput mode raises it substantially).
- Set **visibility timeout ≥ max processing time** (e.g. 6× typical), or extend it while working.

### SNS + SQS fan-out

```
Order service ─▶ [SNS topic: orders] ─┬─▶ [SQS: billing]   ─▶ billing workers
                                       ├─▶ [SQS: shipping]  ─▶ shipping workers
                                       └─▶ [SQS: analytics] ─▶ loader
```
Each subscriber has its **own queue**, so a slow analytics consumer doesn't affect billing. (EventBridge adds content-based routing rules and schema registry.)

### Kafka log

```
Topic "orders" with 6 partitions, replication factor 3 (each partition: 1 leader + 2 followers on different brokers/AZs)

Producer: key = customer_id → hash(key) % 6 → partition 2   (same customer → same partition → ordered)

Partition 2: [0][1][2][3][4][5][6] ... newest
                       ▲             ▲
     group "billing" offset=3       group "analytics" offset=6   (independent progress)

Consumer group "billing": 3 consumers → each owns 2 partitions
   add a 7th consumer to a 6-partition topic → it sits IDLE
```

- **Ordering:** only **within a partition**. Choose the key so things that must be ordered share it.
- **Parallelism ceiling** = number of partitions. Plan partitions for peak consumer parallelism (you can add partitions later but that changes key → partition mapping).
- **Durability:** `acks=all` + `min.insync.replicas=2` with replication factor 3 → a write is confirmed only when on ≥2 brokers. Rack/AZ-aware replica placement.
- **Rebalance:** when consumers join/leave, partitions are reassigned; frequent rebalances stall consumption (use cooperative rebalancing, static membership).
- **Retention:** messages stay after being read (unlike queues), so new consumers can replay. **Log compaction** keeps the latest value per key (good for state/changelog topics).
- **Tiered storage:** old segments moved to object storage for long retention cheaply.

### Delivery semantics

| Semantics | How | Risk | Use |
|---|---|---|---|
| **At-most-once** | Ack before processing | Message lost if consumer crashes | Metrics where loss is OK |
| **At-least-once** ✅ | Process, then ack | Duplicates | **Default**: pair with idempotent consumers |
| **Exactly-once** | Kafka transactions (read-process-write within Kafka), or at-least-once + idempotent writes/dedup | Only within boundaries; side-effects outside (emails, payments) are still at-least-once | Stream processing pipelines |

### Backpressure

When consumers can't keep up, something must give:
- **Queue grows** (buffer) → watch **age of oldest message** and **lag**; scale consumers (KEDA, topic 12).
- **Bound the buffer** → producers get errors/slowdown (backpressure propagates upstream) rather than infinite growth.
- **Shed/sample** low-value messages.
- Unbounded buffers hide problems until retention expires and **data is silently lost** (SQS 14-day max; Kafka retention deletes unread data).

### Sizing Kafka (worked)

Requirements: 200 MB/s peak writes, 3 consumer groups each reading everything, replication factor 3, retention 3 days.

```
Writes to disk (with RF=3):   200 × 3 = 600 MB/s across the cluster
Replication network:          200 × 2 = 400 MB/s between brokers
Reads by consumers:           200 × 3 groups = 600 MB/s (mostly from page cache)
Storage:                      200 MB/s × 86,400 × 3 days × 3 replicas ≈ 155 TB (+ ~20% headroom)
If one broker comfortably handles ~100 MB/s in + replication + reads → ~9–12 brokers,
  across 3 AZs, each with ~15–18 TB disks. (Or tiered storage to cut local disk.)
Partitions: target ~10 MB/s per partition and consumer → 200/10 = 20 → choose 24–48 partitions.
```

Note: cross-AZ replication traffic is billed in AWS; for Kafka it can be a major cost (topic 43): consider follower fetching (consumers read from same-AZ replica) and managed options.

---

## Options and trade-offs

| Need | Choose | Why |
|---|---|---|
| Background jobs, simple decoupling | **SQS** (or RabbitMQ) | Zero ops, scales automatically, per-message ack, DLQ built in |
| One event → many independent consumers | **SNS → SQS per consumer** or EventBridge | Isolation per consumer |
| Strict per-key ordering + high throughput + replay | **Kafka/Kinesis** | Partitioned log, retention |
| Event sourcing / CDC / stream processing | **Kafka** (+ Debezium for CDC, Flink/Kafka Streams) | Log semantics, ecosystem |
| Delayed jobs, scheduling | SQS delay (≤15 min), Cloud Tasks, EventBridge Scheduler, DB-backed scheduler (topic 56) | |
| Very low latency messaging, request-reply | NATS, gRPC streaming | |

| | SQS | Kafka (MSK / self) |
|---|---|---|
| Ops | None | Significant (brokers, partitions, upgrades, rebalancing) unless managed/serverless |
| Ordering | FIFO per group | Per partition |
| Replay | No (once deleted, gone) | Yes, within retention |
| Throughput | Very high (standard) | Very high, with planning |
| Consumer scaling | Unlimited consumers | ≤ partitions per group |
| Per-message retry/DLQ | Built in | Manual (retry topics, DLQ topic) |
| Cost model | Per request | Per broker-hour/storage (or per throughput serverless) |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Consumer lag grows** | Delayed emails/jobs; eventually data expires unread | Lag, age of oldest message alerts | Scale consumers (≤ partitions), optimize processing, add partitions with care |
| **Poison message** | One message fails forever, blocks a partition (Kafka) or loops (SQS) | Same message ID retried; receive count | DLQ after N attempts; Kafka: skip to DLQ topic and alert |
| **Duplicates** | Double emails/charges | Duplicate effects | Idempotent consumers, dedup keys (topic 22) |
| **Visibility timeout < processing time** | Same message processed concurrently by two workers | Duplicate processing logs | Increase timeout / heartbeat extend |
| **Ordering broken** | Updates applied out of order (status "shipped" then "paid") | Data anomalies | Partition by entity key; FIFO groups; version checks in consumer |
| **Rebalance storms** | Consumption stops repeatedly | Rebalance rate metric | Cooperative rebalancing, static membership, tune session/poll timeouts |
| **Broker disk full / under-replicated partitions** | Producer errors, durability risk | `UnderReplicatedPartitions`, disk alerts | Capacity planning, retention, tiered storage |
| **DLQ ignored** | Silent data loss of business events | DLQ depth > 0 alert | Alert on DLQ, runbook to inspect and redrive |
| **Producer can't publish** (broker down) | Lost events if not handled | Publish error rate | Retries, local buffering, **outbox pattern** (topic 22) |

**3 AM:** "Order confirmation emails are 2 hours late." Check queue age/lag for the email consumer. If consumers are healthy but slow: scale out. If consumers are erroring: check the DLQ and logs for a poison message or a failing downstream (email provider rate limit → back off). Communicate the delay; don't purge the queue.

---

## Where you'll use it

- **CI/CD:** build queues (Jenkins build queue, GitHub Actions job queue) are work queues; runner autoscaling on queue length (topic 47).
- **Logging pipelines:** Kafka between log agents and the index (topic 49).
- **Webhooks:** receive → enqueue → process asynchronously (topic 06).
- **Kubernetes:** KEDA scales on SQS/Kafka lag ([K8s 48](../../CKA-Certification-Course-2025/notes/48-metrics-server-and-hpa.md)).
- **Terraform:** `aws_sqs_queue` with `redrive_policy` (DLQ), `aws_sns_topic_subscription`, `aws_msk_cluster`.

---

## Design exercise

**Prompt:** When a user signs up, the system must: create the account (synchronously), send a welcome email, create a CRM record, start a 14-day trial in billing, and record an analytics event. The CRM API is slow and sometimes down for an hour. Design the messaging.

<details>
<summary><b>Model answer</b></summary>

- **Synchronous:** `POST /signup` writes the user row and an **outbox** row "UserSignedUp" in the same DB transaction (topic 22), returns `201`.
- **Publish:** an outbox relay publishes UserSignedUp to **SNS** (or EventBridge).
- **Fan-out:** separate **SQS queues** for email, CRM, billing, analytics subscribed to the topic. Each consumer is independent: CRM outage only backs up the CRM queue.
- **CRM consumer:** retries with exponential backoff (visibility timeout extended progressively), `maxReceiveCount` sized so that an hour-long outage doesn't push messages to the DLQ too early (or use a delay/retry queue); alert on age of oldest message > 30 min; DLQ with alert and a redrive runbook. Idempotent: CRM upsert by user ID.
- **Billing:** idempotent "start trial if not started" keyed by user ID.
- **Email:** dedup by `(user_id, "welcome")` so redelivery doesn't send twice.
- **Analytics:** could go to Kafka/Kinesis/Firehose for the data platform instead of SQS.
- **Why not Kafka for all?** Volume is small, needs are per-message retries/DLQs and independent consumers → SNS+SQS is simpler with zero ops. Kafka if we later need replay or very high volume.
</details>

---

## Quick recap

- Queues decouple time, speed and knowledge: buffer spikes, survive consumer outages, fan out.
- Work queue (one consumer per message), pub/sub (everyone gets a copy), log (retained, partitioned, replayable).
- At-least-once is the norm → idempotent consumers. Visibility timeouts, retries with backoff, DLQs with alerts.
- Kafka ordering is per partition; consumers per group ≤ partitions; size by MB/s, replication and retention.
- Alert on lag / age of oldest message and on DLQ depth.

### Interview questions

1. **Q: SQS or Kafka?**
   A: SQS for task queues and decoupling: no ops, per-message acks, retries and DLQs built in, unlimited consumers. Kafka when I need per-key ordering at high throughput, multiple independent consumers reading the same stream, replay/retention, CDC or stream processing, and I can afford the operational cost (or use a managed service).

2. **Q: Why would a Kafka consumer group with 20 pods be slower than expected?**
   A: If the topic has fewer than 20 partitions, the extra consumers are idle; parallelism is capped by partitions. Also check for skewed keys (one hot partition), slow processing per message, frequent rebalances, and downstream bottlenecks.

3. **Q: How do you handle a message that always fails?**
   A: Limit retries (with backoff), then move it to a DLQ so it doesn't block others or loop forever; alert on DLQ depth; inspect, fix the bug or data, and redrive. In Kafka, publish it to a DLQ topic and commit the offset so the partition can progress.
