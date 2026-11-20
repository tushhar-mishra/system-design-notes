# 62 · Cheat Sheet

> **Level:** ★ foundation
>
> **What this is:** numbers, formulas, patterns and trade-off tables from the whole path on one page. Numbers are rough, for estimation.
>
> **Part 10 · Interview readiness** · Previous: [61 · Senior scenario interviews](61-senior-scenario-interviews.md) · [Back to INDEX](../INDEX.md)

---

## Interview flow (topic 01)

```
Clarify (5) → Estimate (5) → High-level + request walk (10) → Deep dives (15) → Operate: SLO/alerts/failures/deploy/security/DR/cost (7) → Wrap-up trade-offs (3)
```

## Numbers (topic 03)

| | |
|---|---|
| 1 day | 86,400 s ≈ 10^5 |
| 1 M/day | ≈ 12/s |
| 1 month | ≈ 2.6 M s |
| Peak factor | 2–3× (events 10×) |
| 1 Gbps | 125 MB/s |
| KB/MB/GB/TB/PB | 10^3 / 10^6 / 10^9 / 10^12 / 10^15 |
| RAM ref | ~100 ns |
| Same-AZ RTT | ~0.1–0.5 ms ; cross-AZ ~0.5–2 ms |
| SSD random read | ~20–100 µs |
| HDD seek | ~5–10 ms |
| Cross-continent RTT | ~150–250 ms |
| Light in fibre | ~5 ms per 1,000 km one way |
| New HTTPS connection | ~3 RTT before first byte (TCP + TLS 1.3 + request) |

**Rough per-node capacity:** app 500–5k req/s · nginx/Envoy 10k–50k · Postgres 5k–50k reads/s, 1k–10k writes/s · Redis ~100k ops/s · Kafka broker ~50–100 MB/s · OpenSearch node ~5–20 MB/s indexing · Prometheus ~ a few M active series.

## Availability (topic 23)

| Nines | Per year | Per 30 days |
|---|---|---|
| 99% | 3.65 d | 7.2 h |
| 99.9% | 8.76 h | 43.8 min |
| 99.95% | 4.38 h | 21.9 min |
| 99.99% | 52.6 min | 4.4 min |
| 99.999% | 5.26 min | 26 s |

```
Serial:   A = A1 × A2 × …           Parallel (independent): A = 1 − Π(1 − Ai)
A ≈ MTBF / (MTBF + MTTR)            N+1 per AZ: per_AZ = needed / (AZs − 1)
```

## Formulas

```
Little's Law:     L = λ × W                         (concurrency = rate × time)
Queueing wait:    ∝ ρ / (1 − ρ)                     (target 60–70% at peak)
Cache latency:    h × t_cache + (1 − h) × t_db      (90% → 99% hit = 10× less DB load)
Quorum:           ⌊N/2⌋ + 1 ; reads/writes: R + W > N
Burn rate:        error_rate / (1 − SLO)            (14.4 over 1h = 2% of 30-day budget)
Error budget:     (1 − SLO) × requests              (99.9% of 50 M = 50,000)
HPA:              desired = ceil(current × metric / target)
Kafka:            max consumers per group = partitions
Retry amplification: attempts^layers                (4 attempts × 3 layers = 64×)
```

## Burn-rate alerts (99.9% / 30 d) (topic 29)

| Action | Burn | Long / short window |
|---|---|---|
| Page | 14.4 | 1 h / 5 m |
| Page | 6 | 6 h / 30 m |
| Ticket | 1 | 3 d / 6 h |

## Building blocks: when to use (topics 02–22)

| Need | Use |
|---|---|
| Spread load, remove dead servers | Load balancer (L7 for HTTP/gRPC, L4 for TCP/static IPs/huge connection counts) |
| Global users, static content, DDoS | CDN + WAF |
| Hot read data | Cache-aside Redis + TTL jitter + delete-on-write |
| Relations, transactions | Postgres/MySQL (managed, Multi-AZ) |
| Key access at huge scale | DynamoDB/Cassandra (design keys from queries) |
| Files, backups, logs, artifacts | Object storage + lifecycle |
| Slow work, spikes, decoupling | Queue (SQS) / fan-out (SNS→SQS) |
| Ordered stream, replay, many consumers | Kafka/Kinesis |
| Reliable events from DB changes | Outbox / CDC + idempotent consumers |
| Cross-service transaction | Saga with compensations |
| Leader / lock / config consensus | etcd/ZooKeeper/Consul (3 or 5 nodes, 3 AZs) + fencing |

## Resilience toolkit (topic 24)

`Timeouts (outer > inner)` · `Retries: idempotent only, exp backoff + full jitter, one layer, budget` · `Circuit breaker (closed/open/half-open)` · `Bulkheads` · `Load shedding (503 early, prioritize)` · `Rate limit (429 + Retry-After)` · `Graceful degradation` · `Shallow LB health checks` · `Static stability`

## Consistency (topic 19)

Strong at decision points (money, stock, locks) · read-your-writes for the user's own data · eventual for catalogs, counters, search, feeds. CAP: during partition choose C or A. PACELC: else latency vs consistency.

## Deployment strategies (topic 34)

| Strategy | Blast radius | Rollback | Cost |
|---|---|---|---|
| Recreate | 100% + downtime | Slow | None |
| Rolling | Growing | Medium | Surge |
| Blue/green | 100% at switch | Instant | 2× during deploy |
| Canary + analysis | Small % | Fast/auto | Small |
| Feature flags | Chosen segment | Instant | — |

Zero 5xx: readiness, `maxUnavailable: 0`, preStop sleep, SIGTERM drain, deregistration delay, keep-alive > LB idle timeout, PDBs. DB: expand → dual write → backfill → switch reads → stop old writes → contract.

## DR (topic 41)

| Strategy | RPO | RTO | Cost |
|---|---|---|---|
| Backup & restore | hours | hours–day | lowest |
| Pilot light | sec–min | 30 min–hours | low–med |
| Warm standby | seconds | minutes | med–high |
| Active-active | ~0 | ~0–min | highest |

Backups: PITR + snapshots, 3-2-1-1-0, immutable (Object Lock / Vault Lock), separate account, **tested restores**.

## Security (topic 40)

SSO + MFA · workload identity · OIDC for CI · least privilege + SCPs · private subnets + SG references + default-deny NetworkPolicies + egress control · TLS/mTLS + KMS · WAF/DDoS · signed images + SBOM + admission policies · audit logs in separate account · IMDSv2 hop limit 1.

## Cost (topic 43)

Hidden: NAT processing · cross-AZ ($0.01/GB each way) · cross-region · egress · log/metric ingestion · idle/over-requested resources. Levers: rightsize, autoscale, Spot, Savings Plans on baseline, Graviton, lifecycle, VPC endpoints, sampling. Track unit cost.

## K8s platform (topic 38)

Clusters per env × region (+ compliance/untrusted) · namespace tenancy: RBAC, ResourceQuota, LimitRange, NetworkPolicy, Pod Security restricted, policy engine, workload identity · node pools: system / general / Spot / GPU · add-ons via GitOps bundle · staged upgrades with deprecated-API checks.

## IaC at scale (topic 36)

State per env × account/region × layer × owner · versioned opinionated modules · plan on PR + policy on plan + review → apply saved plan via OIDC · one run per state · nightly drift detection · landing zone with SCPs.

## Troubleshooting (topic 58)

`Impact & scope → What changed → Mitigate → Follow the request → RED/USE → Hypothesis → Confirm & prevent` · 502 bad upstream / 503 none healthy / 504 slow upstream · Pending pods → `kubectl describe` events.

## Design review red flags (topic 59)

Single AZ · no backups or "replication is backup" · cache as source of truth · health check hits DB · sync chains of services · shared DB across services · cron on one VM · laptop deploys · secrets in Git · retries without backoff · no SLOs/alerts · multi-region without a data plan.
