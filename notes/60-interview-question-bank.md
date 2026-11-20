# 60 · Interview Question Bank

> **Level:** ★★ intermediate
>
> **What this is:** the common system design questions for DevOps/SRE/Platform interviews, grouped by topic, each with a **short strong answer** (the 30–60 second version) and a pointer to the topic with the full story. Use it for revision: cover the answer, say yours out loud, then compare.
>
> **You need:** topics 01–59.
>
> **Part 10 · Interview readiness** · Previous: [59 · Reviewing designs](59-reviewing-designs.md) · Next: [61 · Senior scenario interviews](61-senior-scenario-interviews.md) · [Back to INDEX](../INDEX.md)

---

## How to answer any question (the 4-beat answer)

```
1. Define it in one line            "A circuit breaker stops calling a failing dependency..."
2. Why it exists / the problem      "...so threads don't pile up waiting and the dependency can recover"
3. How / trade-offs                 "States closed/open/half-open; risk: false trips; tune thresholds"
4. Real example                     "We added one around the fraud API after it caused a cascading outage"
```

---

## A. Approach and estimation (topics 01–03)

1. **How do you start a system design interview?**
   Clarify functional and non-functional requirements (scale, latency, availability, durability, consistency, security, cost), state assumptions with numbers, estimate, then draw the simplest design and deep-dive on the hardest parts, finishing with operations, failures and trade-offs.

2. **How many requests per second is 50 million requests per day?**
   ~50 M / 10^5 ≈ 500/s average; peak ~2–3× → ~1,000–1,500/s.

3. **What's the difference between latency and throughput?**
   Latency is time per request; throughput is requests (or bytes) per second. They're linked by concurrency (Little's Law: concurrency = throughput × latency).

4. **Why plan for peak plus N+1?**
   Average capacity fails every peak; and losing one AZ must not overload the survivors. Example: 12 servers needed at peak, 3 AZs → 6 per AZ → 18 total.

## B. Networking, DNS, LB, edge (topics 04–10, 15)

5. **What happens when you type a URL?**
   DNS resolution (cache → resolver → root/TLD/authoritative), TCP handshake, TLS handshake, HTTP request to CDN edge or LB, routing to a healthy backend, app → cache/DB, response back; mention timing and failure points.

6. **L4 vs L7 load balancer?**
   L4 balances TCP/UDP connections without reading content (fast, any protocol, static IPs); L7 understands HTTP (path/host routing, TLS termination, retries, per-request balancing). gRPC needs L7 or client-side balancing.

7. **Why might DNS failover take longer than the TTL?**
   Health-check detection time plus resolver caching; some clients/JVMs cache longer than TTL. Anycast global LBs fail over faster.

8. **What does a CDN do besides caching?**
   Terminates TLS near users, keeps warm connections to origin, absorbs DDoS, runs WAF/bot rules, serves stale on origin errors, edge compute.

9. **502 vs 503 vs 504?**
   502: bad/reset response from upstream; 503: no healthy upstream or overloaded; 504: upstream too slow (the work may have completed).

10. **What is an API gateway for?**
    Central auth, rate limits/quotas, routing/versioning, validation and observability at the edge of a service platform; not for business logic.

11. **Why is a single NAT gateway a risk?**
    It's AZ-bound; its AZ failing cuts outbound access for all private subnets, and it adds cross-AZ charges. Use one per AZ.

## C. Scaling and caching (topics 11–15)

12. **Vertical vs horizontal scaling?**
    Vertical = bigger machine (simple, limited, SPOF); horizontal = more machines (needs stateless design and an LB, scales further, survives failures).

13. **What makes a service stateless?**
    No request depends on local memory/disk from earlier requests; sessions, files, jobs, schedules are externalized.

14. **Cache-aside vs write-through?**
    Cache-aside: app loads on miss and deletes on write (simple, default). Write-through: writes go to cache and DB together (fresher cache, slower writes).

15. **What is a cache stampede and how do you prevent it?**
    Many requests miss a hot key at once and hammer the DB. Prevent with single-flight/locks, stale-while-revalidate, TTL jitter, early refresh, and DB concurrency limits.

16. **Why can autoscaling fail to save you?**
    It lags minutes (metrics, node launch, app start), may hit quotas/IP limits, and doesn't help if the bottleneck is downstream. Keep headroom, pre-scale, shed load.

## D. Data (topics 16–22)

17. **SQL or NoSQL?**
    Relational by default for relations and transactions; key-value/wide-column when access is by key at a scale/latency one primary can't meet and the access patterns are known.

18. **Replication vs backup?**
    Replication keeps live copies for availability and read scale but copies mistakes instantly; backups/PITR provide point-in-time recovery, ideally immutable and isolated.

19. **Sync vs async replication?**
    Sync: RPO 0, slower writes, availability depends on the replica. Async: fast, RPO = lag. Common: sync across AZs, async across regions.

20. **How do you choose a shard key?**
    High cardinality, even load, present in most queries, keeps transactions on one shard (e.g. tenant_id); watch hot keys and monotonic keys.

21. **Explain CAP.**
    During a network partition, choose consistency (refuse some requests) or availability (serve possibly stale/conflicting data). PACELC adds latency vs consistency when there's no partition.

22. **What's read-your-writes and how do you provide it with replicas?**
    Users always see their own writes; route a user's reads to the primary briefly after a write or for their own data.

23. **Queue vs pub/sub vs log?**
    Queue: each message to one consumer (work distribution). Pub/sub: each subscriber gets a copy (fan-out). Log (Kafka): retained, partitioned, replayable, consumer groups track offsets.

24. **What is the outbox pattern?**
    Write the business change and the event in the same DB transaction; a relay/CDC publishes the event later. Solves the dual-write problem; consumers must be idempotent.

25. **What is a saga?**
    A multi-service transaction as a sequence of local transactions with compensating actions on failure; choreographed by events or orchestrated by a coordinator/workflow engine.

## E. Reliability (topics 23–26, 41–42, 44)

26. **What's 99.9% in downtime?**
    ~43.8 minutes per month, ~8.76 hours per year.

27. **Two services at 99.9% in series?**
    ~99.8%. Dependencies multiply availability down; redundancy (independent) multiplies failure probability down.

28. **Retries: what can go wrong?**
    Retry storms amplify load (multiplied across layers), duplicates for non-idempotent ops. Use backoff + jitter, retry at one layer, budgets, idempotency keys.

29. **Circuit breaker vs bulkhead?**
    Circuit breaker stops calls to a failing dependency (fail fast, let it recover); bulkhead isolates resources (pools) so one dependency can't exhaust everything.

30. **Why 3 or 5 nodes for etcd?**
    Quorum is a majority; odd sizes maximize fault tolerance per node (3 tolerates 1, 5 tolerates 2); spread across 3 AZs.

31. **RPO vs RTO?**
    RPO: acceptable data loss (time). RTO: acceptable downtime. They choose the DR strategy: backup/restore, pilot light, warm standby, active-active.

32. **Active-active multi-region: hardest part?**
    Data: write conflicts and cross-region latency → home regions per record, multi-writer with conflict resolution, or global consensus DB; plus regional independence and failover capacity.

33. **What is chaos engineering?**
    Controlled experiments injecting failures to verify resilience: steady state, hypothesis, small blast radius, abort conditions, learn and fix.

## F. Observability and SRE (topics 27–31)

34. **Metrics vs logs vs traces?**
    Metrics for alerting and trends (cheap, aggregated); logs for detailed events; traces for following one request across services to find where time/errors happen.

35. **Why percentiles, not averages?**
    Averages hide tail latency; p99 reflects the worst-served users, and fan-out requests hit tails often.

36. **SLI vs SLO vs SLA?**
    SLI: measured good/valid ratio. SLO: internal target. SLA: contractual promise with penalties (looser than SLO).

37. **What is an error budget?**
    1 − SLO: the allowed unreliability; spending decisions (ship vs stabilize) follow an agreed policy.

38. **What should page a human?**
    Urgent, actionable, user-impacting conditions, mainly fast error-budget burn; causes go to dashboards/tickets.

39. **What do you do first in an incident?**
    Assess impact, declare, assign roles, check what changed, mitigate (rollback, failover, shed load), communicate; root cause later; blameless postmortem.

40. **Little's Law?**
    L = λ × W: items in system = arrival rate × time in system. Used for sizing pools, runners, queues.

## G. Delivery and platform (topics 32–39)

41. **Why build once and promote?**
    So the exact tested artifact runs in prod; promote by digest; rebuilds can differ.

42. **Push vs GitOps?**
    Push: CI applies to targets (simple, needs creds). GitOps: in-cluster agent pulls desired state from Git (no creds in CI, drift detection, audit, easy rollback via revert).

43. **Blue/green vs canary?**
    Blue/green: full new environment, atomic switch, instant switch-back, 2× capacity. Canary: small % first with metric analysis, smaller blast radius, needs traffic splitting and version compatibility.

44. **How do you do zero-downtime DB migrations?**
    Expand/contract: additive changes, dual writes, batched backfills, switch reads, contract later; lock-safe DDL with `lock_timeout`, concurrent indexes.

45. **How do you split Terraform state?**
    By environment, account/region, layer and owner; small blast radius; share outputs via published parameters.

46. **How should CI authenticate to the cloud?**
    OIDC federation to short-lived role credentials scoped by repo/branch/environment; no static keys.

47. **One cluster or many?**
    Separate by environment, region, compliance and trust; share with namespace tenancy (RBAC, quotas, network policies, policy engine) inside.

48. **What is platform engineering?**
    Building self-service internal products (golden paths with guardrails) that reduce developer cognitive load, measured by developer outcomes (DORA, time to first deploy).

## H. Security and cost (topics 37, 40, 43)

49. **How do pods get secrets securely?**
    Workload identity → secret manager (ESO/CSI/Vault agent), KMS encryption, etcd encryption, RBAC; prefer dynamic short-lived credentials.

50. **Top causes of cloud breaches?**
    Leaked/over-privileged credentials and misconfigurations; prevent with SSO+MFA, no long-lived keys, least privilege, guardrails, detection.

51. **What is zero trust?**
    No implicit trust from network location; every request authenticated and authorized by identity and context (mTLS, policies), plus least privilege.

52. **Hidden cloud costs?**
    NAT processing, cross-AZ/region transfer, egress, log/metric ingestion, idle resources, over-requested K8s resources.

53. **How do you reduce cost without hurting reliability?**
    Rightsize from data, autoscale, Spot for tolerant workloads, commitments for the baseline, lifecycle storage, VPC endpoints, sampling telemetry; keep SLO headroom and validate with load tests.

---

## Quick self-test (answer in one breath each)

Nines for 99.95% per month? *(~22 min)* · Quorum for 5 nodes? *(3)* · Why jitter? *(desynchronize retries)* · Cache key too wide effect? *(hit ratio collapses)* · Kafka consumers > partitions? *(idle consumers)* · RPO of async cross-region? *(replication lag)* · Burn rate 14.4 means? *(2% of 30-day budget per hour)* · Expand/contract first step? *(add, don't change/remove)* · Safest default when a feature flag service is unreachable? *(last known ruleset, then code defaults)*.
