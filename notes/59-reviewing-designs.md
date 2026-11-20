# 59 · Reviewing Someone Else's Design

> **Level:** ★★★ advanced
>
> **You will learn:** how a senior engineer **reviews a design document or architecture** (an RFC, a whiteboard proposal, an existing system): the **review checklist** by category (requirements, scale, SPOFs, failure modes, data, consistency, security, operations, deploys, cost, team), the **red flags** that appear again and again, how to **give feedback** that improves the design without blocking the team, and a worked review.
>
> **You need:** topics 01–57.
>
> **Part 10 · Interview readiness** · Previous: [58 · Troubleshooting](58-troubleshooting-at-scale.md) · Next: [60 · Interview question bank](60-interview-question-bank.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Senior engineers spend as much time **reviewing** designs as creating them. Some interviews say: "Here's our architecture. What would you change?" A good review is structured, prioritized, and constructive.

**The 12-question checklist:**
```
 1. Requirements: are functional + non-functional (numbers!) written down?
 2. Scale: is there an estimate? Does each component fit it with headroom?
 3. SPOFs: what single thing, if it dies, stops the user function?
 4. Failure modes: for each dependency — timeout, retry, fallback, what the user sees?
 5. Data: source of truth, consistency model per operation, backups, RPO/RTO, migrations?
 6. State: what's stateful, where, how does it fail over and scale?
 7. Security: identities, least privilege, data protection, edge, secrets, audit?
 8. Operations: SLOs, alerts, dashboards, runbooks, on-call owner?
 9. Delivery: how is it deployed, rolled back, migrated? Config changes?
10. Cost: estimate, main drivers, unit cost?
11. Simplicity: could a simpler design meet the requirements? What can be removed?
12. Evolution: what happens at 10× scale? What's the migration path? What's reversible?
```

**Picture it.** A building inspector with a checklist (fire exits, load-bearing walls, wiring, plumbing) rather than "it looks nice". They also suggest cheaper materials where the plan over-builds.

**The one thing to know.** **Prioritize feedback:** blockers (data loss, security holes, SPOFs for critical paths) vs important (operability, cost) vs nits. And always propose an alternative, not just a problem.

**You've got it if you can:** review a design in 15 minutes and produce a ranked list of 5–8 findings with fixes.

> 📖 **That's the core.** Everything below is the full checklist and a worked review.

---

## What it is

A **design review** checks a proposed (or existing) architecture against its requirements and against known failure patterns before it's expensive to change. It's done in RFC comments, review meetings, production-readiness reviews (PRRs) before launch, and in interviews ("critique this diagram").

## Why it exists

Fixing a design flaw on paper costs minutes; fixing it after launch costs months (data migrations, outages). Reviews also spread knowledge and keep architecture consistent across teams.

## Analogy

The building inspector (Core). Also an editor: the goal is a better text, not proving the author wrong.

---

## How it works inside: the detailed checklist

### Requirements and scale
- Are there numbers: QPS (avg/peak), data size and growth, latency, availability, RPO/RTO? If not: ask before reviewing anything else.
- Does the estimate match the components? (e.g. 100k writes/s into one Postgres primary = red flag; Kafka for 5 events/s = over-engineering.)
- Peak events, seasonality, tenants of very different sizes?

### Availability and failure
- Walk every box: "If this dies?" Identify SPOFs: single NAT, single DB without standby, single Redis, single region dependency, one person.
- Hard dependencies on the critical path: timeouts, retries (with backoff/jitter/budget), circuit breakers, fallbacks defined?
- Health checks: do they check shared dependencies (cascading 503 risk)?
- Capacity for N−1 AZs? Static stability?
- Cross-region synchronous calls on the request path?

### Data
- Source of truth per entity; who writes; consistency per operation (strong at decision points?).
- Replication vs backups (PITR, immutable, separate account), tested restores, RPO/RTO met?
- Dual writes (DB + broker)? → outbox. Idempotency for retries and at-least-once consumers?
- Schema change strategy (expand/contract)? Hot keys/partitions? Growth/archival?

### Security
- AuthN/AuthZ for users and services; least-privilege IAM; no long-lived keys.
- Data classification: PII, encryption at rest/in transit, residency.
- Edge: WAF, rate limits, DDoS; origin protection.
- Secrets management; audit logs; threat model of new trust boundaries.

### Operability
- SLOs and SLIs defined; alerts on symptoms; dashboards; runbooks; ownership in the catalog; on-call.
- Logs/metrics/traces with correlation; cardinality concerns.
- Capacity plan and load test plan.

### Delivery
- Build once, promote; deployment strategy; rollback path (including data); feature flags; config change safety.
- IaC for all infrastructure; state layout and blast radius.

### Cost
- Monthly estimate; top drivers; egress/NAT/cross-AZ; observability cost; unit cost.
- Over-provisioned redundancy relative to tier?

### Simplicity and evolution
- Each component justified by a requirement? (Remove the rest.)
- Fits team skills and existing platform?
- Reversible vs one-way-door decisions identified (database choice, sharding key, public API contracts)?
- What breaks first at 10×?

### Red flags list (memorize)

| Red flag | Why |
|---|---|
| "We'll add monitoring later" | You'll be blind at launch |
| Shared database across services | Coupling, distributed monolith |
| Synchronous chain of 6+ services | Latency and availability multiply down |
| Cache is required for correctness / no fallback | Cache outage = outage, stampedes |
| No backups or "replication is our backup" | Logical disasters unrecoverable |
| Cron on a single VM | SPOF, silent missed runs |
| Secrets in env files/Git | Leaks |
| Manual steps in failover | Slow, error-prone at 3 AM |
| Health check hits the DB | Cascading total outage |
| Retries everywhere with no backoff | Retry storms |
| Unbounded queues/lists/payloads | Hidden latency and memory blow-ups |
| Multi-region "for safety" without data plan | Cost without real resilience |
| Single point of human knowledge | Bus factor |

---

## Options and trade-offs: how to give feedback

| Style | Example | Effect |
|---|---|---|
| ❌ Vague | "This won't scale." | Defensive, no action |
| ❌ Solution-dictating | "Use Kafka." | Misses context |
| ✅ Requirement-linked + question | "At 20k writes/s peak (section 2), a single primary will be ~80% CPU by our load test numbers. How do we handle 2× growth? Options: partition by tenant, or move events to a log." | Constructive, specific |
| ✅ Severity labels | **Blocker / Important / Suggestion / Nit** | Team knows what must change before launch |

Also: acknowledge good decisions, ask questions before asserting, review the doc not the person, and agree on follow-ups with owners.

---

## Failure modes (of reviews)

| Failure | Fix |
|---|---|
| Review too late (after build) | Review at RFC stage; lightweight early reviews |
| Bikeshedding (nits dominate) | Severity labels, focus on blockers first |
| Rubber stamping | Checklist + PRR before launch |
| Reviewer imposes personal preferences | Tie every comment to a requirement or known risk |
| No follow-through | Action items with owners, re-review if blockers |

---

## Where you'll use it

- Interviews: "critique this architecture" and the second half of every design round ("what are the weaknesses of your own design?").
- RFC reviews, production-readiness reviews, postmortems ("what in the design allowed this?").

---

## Design exercise: worked review

**Design under review (summarized):** "Order service v2: EKS in one AZ (to save cross-AZ costs), RDS Postgres single instance with daily snapshots, Redis for sessions and as the **source of truth for inventory counts** (fast!), synchronous calls to email, inventory and payment services on checkout, a nightly cron on one EC2 instance for reconciliation, CI deploys with `kubectl apply` from engineers' laptops, logs on node disks, expected 2,000 orders/min peak, target 99.95%."

<details>
<summary><b>Model review</b></summary>

**Blockers**
1. **Single AZ for EKS and single-instance RDS** → an AZ issue = full outage; can't reach 99.95% (topic 23). Fix: 3 AZs, RDS Multi-AZ (or Aurora); cross-AZ cost is small relative to the risk (topic 43).
2. **Redis as source of truth for inventory** → data loss on failover/eviction, no durability guarantees by default. Fix: inventory in Postgres with conditional updates (`stock = stock - 1 WHERE stock > 0`); Redis only as cache (topics 14, 19).
3. **Backups: daily snapshots only** → RPO 24 h for orders. Fix: PITR enabled (seconds), cross-account copies with Vault Lock, restore tests (topic 41).
4. **Deploys from laptops with kubectl** → no audit, no rollback discipline, credential sprawl. Fix: CI build once + GitOps + canary (topics 32–34).

**Important**
5. **Synchronous email call on checkout** → email outage fails checkout. Fix: outbox + queue for email; keep payment synchronous with timeout, idempotency key, circuit breaker; inventory reservation in the same DB transaction or a saga (topics 21–24).
6. **Cron on one EC2** → SPOF and silent misses. Fix: K8s CronJob / managed scheduler, idempotent, with alerting on missed runs (topic 56).
7. **Logs on node disks** → lost on node replacement, no search. Fix: stdout → log pipeline (topic 49).
8. **No SLOs/alerts mentioned** → add checkout SLI/SLO and burn-rate alerts (topics 28–29).

**Suggestions**
9. Load math: 2,000 orders/min ≈ 33/s → modest; a single Postgres primary is fine; no need for sharding. Good, keep it simple.
10. Sessions in Redis are fine (with replicas); consider signed cookies.
11. Add a cost estimate and unit cost per 1,000 orders.

**Positive:** stateless service on EKS, Redis used for sessions, reasonable scale assumptions.
</details>

---

## Quick recap

- Review with a checklist: requirements & scale, SPOFs & failures, data, security, operability, delivery, cost, simplicity, evolution.
- Know the recurring red flags (shared DBs, sync chains, cache-as-truth, no backups, health checks on DB, laptop deploys, cron on one VM).
- Rank findings: blocker / important / suggestion / nit; link each to a requirement; propose alternatives.
- Praise good choices; prefer the simplest design that meets requirements.
- Apply the same review to your own design at the end of every interview.

### Interview questions

1. **Q: Here's an architecture. What are its biggest risks?**
   A: (Method) Restate requirements and numbers, walk the request path for SPOFs and failure handling, check data durability/consistency and backups, security and identity, operability (SLOs, alerts, ownership), delivery/rollback, and cost; then present the top 3 risks with concrete fixes, ordered by impact on users and data.

2. **Q: How do you disagree with a senior colleague's design in a review?**
   A: Tie the concern to a requirement or evidence (numbers, a past incident), ask questions to understand their constraints, propose an alternative with its trade-offs, and suggest a way to decide (prototype, load test, limited rollout). Keep it about the design and agree on the decision and owner.

3. **Q: What do you check in a production-readiness review before launch?**
   A: SLOs and alerts with runbooks, on-call ownership, load and failure test results, backups with a tested restore, security review and threat model, deployment and rollback procedure, capacity plan with N+1, dashboards, cost estimate, and documentation.
