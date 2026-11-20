# 22 · Async Patterns That Don't Lose Data

> **Level:** ★★★ advanced
>
> **You will learn:** how to build reliable flows on top of at-least-once messaging: **idempotent consumers** (dedup tables, natural idempotency, versions), the **dual-write problem** and the **transactional outbox** pattern, **CDC** (Change Data Capture), the **inbox** pattern, **sagas** (choreography vs orchestration) with **compensating actions**, **retries with backoff and DLQs**, why **exactly-once** is mostly a myth end to end, **reconciliation jobs**, and workflow engines (Temporal, Step Functions).
>
> **You need:** topics 06 (idempotency keys), 13, 16, 21.
>
> **Part 3 · Caching and data** · Previous: [21 · Queues and streaming](21-queues-pubsub-streaming.md) · Next: [23 · Availability and the nines](23-availability-and-the-nines.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Async systems fail in the gaps: between "saved to DB" and "published event", between "processed" and "acked". Three patterns close the gaps:

```
1. Idempotent consumer:  processing the same message twice = same result as once
2. Transactional outbox: write business data + event in ONE DB transaction; a relay publishes later
3. Saga:                 a multi-service workflow as a chain of local transactions,
                         each with a compensating "undo" if a later step fails
```

**Picture it.** Sending a parcel with a receipt book. You write the receipt **in the same moment** you pack the box (outbox), so you never ship without a record or record without shipping. The receiver checks the parcel number against their log and ignores duplicates (idempotency). If the trip is cancelled midway, each step has an undo: refund, return to sender (saga compensation).

**The one thing to know.** You can't atomically "write to the DB **and** send to Kafka" with two separate calls. Either can fail after the other succeeds (the **dual-write problem**). The outbox pattern turns it into one local transaction.

**Try it now:** find a place in a codebase you know that does `db.save(...)` then `publisher.send(...)`. Ask: what if the process dies between the two lines?

**You've got it if you can:** implement an idempotent consumer, explain the outbox, and design a saga with compensations.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Pattern | Problem it solves |
|---|---|
| **Idempotent consumer** | Duplicate messages (at-least-once delivery, retries) |
| **Transactional outbox** | Dual write: DB updated but event not published (or vice versa) |
| **CDC** (Change Data Capture) | Stream DB changes as events by reading the DB's log (Debezium, DMS, DynamoDB Streams) |
| **Inbox** | Consumer side dedup + ordering: store incoming message IDs in the same transaction as the effect |
| **Saga** | Business transactions spanning services without distributed transactions |
| **Compensating action** | The "undo" for a completed saga step (refund, release stock) |
| **Reconciliation** | Periodic job comparing systems to find and fix drift (the safety net) |
| **Workflow engine** | Durable orchestration with state, retries, timers (Temporal, AWS Step Functions, Azure Durable Functions, GCP Workflows) |

**Two-phase commit (2PC)** = a coordinator asks all participants "prepare?", then "commit!". It gives atomicity across systems but blocks if the coordinator dies mid-way and needs all participants to support it. Rarely used across microservices; sagas replace it.

---

## Why it exists

Topic 21 told you brokers deliver **at least once** and topic 13 told you microservices **can't share transactions**. Without these patterns you get: double charges, orders without payments, emails for orders that never existed, stock that never gets released. These bugs are rare, silent and expensive, exactly the kind seniors are expected to design out.

---

## Analogy

Parcel and receipts (Core). Saga extra: booking a trip. Book flight → book hotel → book car. If the car booking fails, you don't "roll back time"; you **cancel the hotel and the flight** (compensations), and maybe pay a cancellation fee (compensations aren't always perfect).

---

## How it works inside

### 1. Idempotent consumers

Ways to make processing safe to repeat:

| Technique | How | Example |
|---|---|---|
| **Natural idempotency** | The operation is already idempotent | `SET status='shipped'`, upsert by key, `PUT` |
| **Dedup table (processed IDs)** | Record message ID in the **same transaction** as the effect; unique constraint rejects repeats | `INSERT INTO processed(msg_id) ...` + business update, one transaction |
| **Idempotency key at the API** | Store key → result (topic 06) | Payment API |
| **Conditional / versioned writes** | Apply only if version matches / state allows | `UPDATE order SET status='paid', v=v+1 WHERE id=? AND status='pending'` |
| **Deterministic IDs** | Derive output IDs from input | Thumbnail key = `hash(photo_id + size)`; rewrite is harmless |

```sql
BEGIN;
  INSERT INTO processed_messages(message_id) VALUES ('m-123');   -- fails on duplicate → ROLLBACK, ack message
  UPDATE accounts SET balance = balance - 100 WHERE id = 7;
COMMIT;
-- then ack the message
```

If the consumer crashes after COMMIT but before ack, the message is redelivered; the INSERT fails on the unique key → skip → ack. Correct.

**Side effects outside your DB** (sending an email, calling a payment provider): pass an idempotency key to the provider if it supports one; otherwise record "about to send" + "sent" states and accept rare duplicates or check before sending.

### 2. Transactional outbox

```
❌ Dual write:
   db.commit(order)        ✅
   kafka.send(OrderPlaced) ✗ (crash / broker down) → order exists, nobody knows

✅ Outbox:
   BEGIN
     INSERT INTO orders ...
     INSERT INTO outbox(id, topic, payload, created_at) ...
   COMMIT                                   ← atomic: both or neither
   Relay (separate process):
     loop: SELECT unpublished FROM outbox ORDER BY id LIMIT 100
           publish to broker (at-least-once)
           mark published (or delete)
   OR: CDC (Debezium) reads the DB's WAL for the outbox table and publishes → no polling
```

The relay may publish a row twice (crash after publish, before marking) → consumers must be idempotent (pattern 1). That's fine; the guarantee is **never lost, maybe duplicated**.

Clean up the outbox table (delete published rows after N days) or it grows forever.

### 3. CDC (Change Data Capture)

Reads the database's replication log (Postgres logical replication, MySQL binlog) and emits a change event per row change. Uses: outbox relay, feeding search indexes/caches/warehouses, strangler-fig migrations (topic 13). Watch: replication slots in Postgres keep WAL until consumed → **a stuck CDC connector can fill the DB disk** (alert on slot lag).

### 4. Sagas

A **saga** = a sequence of local transactions T1, T2, T3..., each publishing an event/command for the next. If Tn fails, run compensations C(n-1)...C1.

**Example: place order**

```
T1 Orders:   create order (PENDING)
T2 Payments: authorize card                 C2: void authorization / refund
T3 Stock:    reserve items                  C3: release reservation
T4 Orders:   mark CONFIRMED
If T3 fails (out of stock): run C2 (void auth), then mark order CANCELLED.
```

**Choreography** (events, no central brain):
```
Orders ─OrderCreated─▶ Payments ─PaymentAuthorized─▶ Stock ─StockReserved─▶ Orders (confirm)
                                               Stock ─StockFailed─▶ Payments (void) ─▶ Orders (cancel)
```
+ simple for 2–4 steps, loosely coupled. − Hard to see the whole flow; cyclic dependencies; debugging needs tracing.

**Orchestration** (a coordinator):
```
[Order saga orchestrator / workflow engine]
   → Payments.authorize → ok
   → Stock.reserve      → fail
   → Payments.void      → ok
   → Orders.cancel
```
+ Flow in one place, explicit state, timeouts, retries, visibility. − The orchestrator is a component to run (or use Temporal/Step Functions).

**Saga rules:**
- Every step and compensation must be **idempotent** and **retryable**.
- Compensations are **semantic** (refund), not physical rollback; they may not perfectly undo (emails already sent).
- Model intermediate states explicitly (PENDING, CONFIRMED, CANCELLED) so users and ops can see them.
- Add **timeouts**: a step that never replies must eventually trigger compensation.
- **Isolation is weak:** other transactions can see intermediate states (a "reserved" item shows as unavailable). Use semantic locks/states.

### 5. Retries done right

```
attempt 1 → fail → wait 1 s (+ jitter) → attempt 2 → wait 2 s → 4 s → 8 s ... → max N → DLQ + alert
```
- **Exponential backoff with jitter** (randomness spreads retries, topic 24).
- Distinguish **retryable** (timeout, 503, throttling) from **non-retryable** (400 validation error → straight to DLQ).
- Total retry window should cover typical dependency outages (e.g. an hour for a CRM) without blocking other work: use **delayed retry queues** rather than sleeping in the consumer.

### 6. "Exactly-once"

- Inside Kafka (consume → process → produce to Kafka), **Kafka transactions** + idempotent producers give exactly-once processing.
- End to end with external side effects, you get **effectively-once** = at-least-once delivery + idempotent effects. Say this in interviews.

### 7. Reconciliation: the safety net

A scheduled job compares sources of truth: payment provider's settlement report vs our payments table; orders CONFIRMED vs shipments created. It catches whatever slipped through (bugs, manual DB edits, provider issues), alerts, and auto-fixes safe cases. Every money-moving system has one.

---

## Options and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| Publishing events reliably | Dual write (❌) / outbox + polling relay / outbox + CDC / event sourcing | Outbox + polling is simple; CDC scales and has low latency but adds Debezium/Kafka Connect to run |
| Saga style | Choreography / orchestration / workflow engine | ≤3–4 steps, few teams → choreography. Complex flows, timers, human steps → orchestration (Temporal, Step Functions) |
| Dedup storage | Same DB as effect (✅ atomic) / Redis with TTL (fast, not atomic) | Prefer same transaction as the effect |
| Cross-service atomicity | 2PC / saga | Saga, unless all participants are one DB |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Dual write** | Missing events or phantom events | Reconciliation mismatches | Outbox / CDC |
| **Non-idempotent consumer** | Duplicates on redelivery | Duplicate side effects | Dedup table, conditional writes |
| **Outbox relay stuck** | Events delayed; outbox table grows | Oldest unpublished row age alert | Relay HA (leader election, topic 26), alert |
| **CDC slot lag** | DB disk fills with WAL | Replication slot lag | Alert; drop/recreate slot if connector dead (after assessing data impact) |
| **Saga stuck mid-way** | Orders PENDING forever | Count of sagas in non-terminal state > N min | Timeouts → compensation; workflow engine visibility |
| **Compensation fails** | Money taken, order cancelled, no refund | Reconciliation | Retries + DLQ + human runbook; compensation must be idempotent |
| **Out-of-order events** | Older update overwrites newer | Version anomalies | Partition by entity, include versions, ignore older versions |
| **Retry storm** | Consumers hammer a failing dependency | Dependency error rate, retry counts | Backoff + jitter, circuit breaker (topic 24), delay queues |

**3 AM:** "Customers charged but orders show cancelled." Saga compensation (refund) failing. Stop the bleeding: pause the step causing failures if possible; check the compensation DLQ; run refunds via the runbook; communicate. Postmortem: reconciliation alert should have caught it within minutes.

---

## Where you'll use it

- **CI/CD platforms:** a "deploy" is a saga: build → push image → update GitOps repo → sync → verify; compensation = rollback (topic 33–34).
- **Infrastructure provisioning workflows** (self-service platform, topic 53): Terraform apply → DNS → monitoring registration, with compensation on failure.
- **Webhook processing:** idempotent by delivery ID (GitHub sends `X-GitHub-Delivery`).
- **Ansible/Terraform:** idempotency is the same core idea ([Ansible 27 · idempotency in depth](../../Ansible/notes/27-idempotency-in-depth.md)).
- **Job scheduler** case study (topic 56).

---

## Design exercise

**Prompt:** A self-service "create database" feature in your internal platform: user requests a DB → platform runs Terraform to create an RDS instance (10 min) → creates a Vault secret with credentials → registers the DB in monitoring → notifies the user in Slack. Any step can fail. Design it so nothing is half-created silently.

<details>
<summary><b>Model answer</b></summary>

- **API:** `POST /databases` with an idempotency key; writes a `requests` row (state REQUESTED) + outbox event in one transaction; returns `202 Accepted` with a request ID.
- **Orchestration:** a workflow engine (Temporal or Step Functions) runs the saga, since steps are long (10-minute Terraform), need timers and visibility:
  1. `terraform apply` for this DB's own state (idempotent: re-running converges). On failure after retries: compensation = `terraform destroy` of that state.
  2. Create Vault database secrets engine role (idempotent upsert). Compensation: delete role.
  3. Register in monitoring (upsert by DB ID). Compensation: deregister.
  4. Notify Slack (dedup by request ID; failure here doesn't trigger compensation, just retries/alert).
  5. Mark request READY.
- **States** visible in the portal: REQUESTED → PROVISIONING → CONFIGURING → READY / FAILED (with reason) / ROLLED_BACK.
- **Timeouts:** step 1 timeout 30 min → compensation. Stuck-request alert if any request is non-terminal for > 1 h.
- **Reconciliation:** nightly job compares RDS instances tagged `managed-by=platform` vs `requests` table and Terraform states; flags orphans (created but not recorded) and ghosts (recorded but missing).
- Trade-off: a workflow engine is another system to run, but hand-rolled sagas for long multi-step infra workflows are where orphaned cloud resources (and surprise bills) come from.
</details>

---

## Quick recap

- At-least-once is reality → make every consumer idempotent (dedup in the same transaction, conditional writes, deterministic IDs).
- Never dual-write; use the transactional outbox (polling relay or CDC).
- Sagas replace distributed transactions: local steps + idempotent compensations + timeouts + visible states.
- Retries: backoff + jitter, retryable vs not, DLQ + alert.
- "Exactly-once" end to end = at-least-once + idempotency; reconciliation jobs are the safety net.

### Interview questions

1. **Q: What is the dual-write problem and how do you solve it?**
   A: Writing to two systems (DB and broker) with separate calls can't be atomic: a crash between them leaves them inconsistent. The transactional outbox writes the business change and the event to the same database in one transaction; a relay or CDC publishes the outbox to the broker afterwards. Delivery becomes at-least-once, so consumers are idempotent.

2. **Q: Choreography or orchestration for a saga?**
   A: Choreography for short flows owned by few teams, where loose coupling matters and events are natural. Orchestration when the flow is long or complex, has timers and human steps, or needs clear visibility and ownership; a workflow engine gives durable state, retries and timeouts. Many companies end up orchestrating anything with more than 3–4 steps.

3. **Q: Kafka offers exactly-once. Can you guarantee a customer is never emailed twice?**
   A: Kafka's exactly-once covers reads and writes within Kafka. Sending an email is an external side effect, so delivery is at-least-once. To avoid duplicates I record the send keyed by `(user, email_type)` before/after sending, or use a provider idempotency key, accepting a tiny window if the provider doesn't support one.
