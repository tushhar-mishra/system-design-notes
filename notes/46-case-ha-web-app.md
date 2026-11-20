# 46 · Case Study: A Highly Available Web App on Cloud

> **Level:** ★★★ advanced
>
> **You will practise:** the most common DevOps design prompt: a **multi-AZ, three-tier** web application on AWS (with GCP/Azure equivalents), from DNS to database, with **autoscaling, caching, CDN, zero-downtime deploys, IaC, CI/CD, monitoring, DR and cost**, and the **"3 AM walkthrough"** of each failure.
>
> **You need:** topics 01–43.
>
> **Part 9 · Case studies** · Previous: [45 · URL shortener](45-case-url-shortener.md) · Next: [47 · CI/CD platform](47-case-cicd-platform.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design a highly available e-commerce web application on AWS."

| Question | Assumed answer |
|---|---|
| Users / traffic? | 2 M daily active users; peak 3,000 req/s; sale events up to 4× |
| Availability target? | 99.95% (≈ 22 min/month); checkout is critical |
| Latency? | p99 < 500 ms for pages, < 300 ms for API |
| Data loss? | Orders: RPO ≈ 0 for AZ failure, ≤ 5 min for region failure |
| RTO? | AZ failure: automatic, < 5 min. Region failure: < 4 h (business accepts) |
| Users' location? | Mostly India, some global |
| Existing stack? | Containerized app (Java + React), team knows Kubernetes, Terraform, GitHub Actions |
| Compliance? | Card payments via a PCI-compliant provider (we never store card numbers) |

---

## 2. Estimation

```
Peak 3,000 req/s (sale 12,000 req/s). Average ~1,000 req/s.
API pod capacity (load-tested): ~300 req/s at p99 < 300 ms → 70% → 210 req/s planning capacity
Normal peak: 3,000/210 ≈ 15 pods → N+1 over 3 AZs: 15/2 ≈ 8 per AZ → 24 pods
Sale peak: 12,000/210 ≈ 58 → 29 per AZ → ~87 pods (pre-scale)

DB: ~3 queries/request, 85% cache hit on reads → ~1,500–6,000 queries/s → one Aurora writer + 2 readers
Orders: 200,000/day × 2 KB = 400 MB/day → ~150 GB/year (small)
Static assets/images: 2 TB in S3; egress via CDN ≈ 2 M users × 3 MB/day = 6 TB/day
```
**Conclusion:** the stateless tier and DB scale comfortably; the important design points are **multi-AZ redundancy everywhere, CDN offload, safe deploys and the data layer's failover/DR**.

---

## 3. High-level design

```
                     Route 53 (alias, health checks)          ACM certs        AWS WAF + Shield
Users ──▶ CloudFront (static from S3 via OAC; dynamic → origin)──────────────────────────────┐
                                                                                             ▼
┌────────────────────────── VPC 10.10.0.0/16, ap-south-1, 3 AZs ──────────────────────────────┐
│ public subnets:   ALB (all 3 AZs)                    NAT GW ×3 (one per AZ)                   │
│ private-app:      EKS managed control plane; Karpenter node pools spread over 3 AZs           │
│                   ├─ web (SSR/BFF) pods      ├─ api pods        ├─ worker pods (KEDA on SQS) │
│ private-data:     Aurora PostgreSQL (writer AZ-a, readers AZ-b/c)                             │
│                   ElastiCache Redis (cluster mode, primaries+replicas across AZs)             │
│ AWS services via VPC endpoints: S3, ECR, STS, Secrets Manager, SQS, CloudWatch               │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
Async: api → SQS (order-events) → workers (emails, invoices, search indexing)
Ops:   GitHub Actions (OIDC) → ECR (signed images) → Argo CD (GitOps) → EKS ; Terraform for infra
Obs:   Prometheus/Grafana (or AMP/AMG), Loki/CloudWatch logs, OTel traces, synthetic checks
DR:    Aurora Global DB secondary + S3 CRR + ECR replication to ap-southeast-1 (pilot light)
```

**Request flow (product page):**
1. DNS → CloudFront edge near the user (TLS terminated at edge).
2. Static assets (JS/CSS/images) served from edge cache (hashed filenames, 1-year TTL).
3. HTML/API → WAF rules → origin ALB (only accepts CloudFront, via a secret header + managed prefix list).
4. ALB (IP-mode targets) → web/api pod in any AZ.
5. Pod reads product from Redis (hit) or Aurora reader (miss), fills cache.
6. Response back through ALB → CloudFront → user.

**Checkout flow:** api → payment provider (idempotency key, timeout 3 s, circuit breaker) → Aurora writer transaction (order + outbox row) → outbox relay → SQS → workers send confirmation email.

---

## 4. Deep dives

### 4.1 Stateless tier on EKS
- Deployments with `topologySpreadConstraints` across zones, PDBs (`minAvailable: 70%`), readiness/liveness probes, preStop + graceful shutdown (topic 34).
- HPA on requests per pod (custom metric) with min = N+1 sizing; Karpenter with on-demand for api/web, Spot allowed for workers.
- Sessions in Redis (or signed cookies); no local state.

### 4.2 Data layer
- **Aurora PostgreSQL**: storage replicated 6 ways across 3 AZs; writer failover ~30 s; reader endpoint for lag-tolerant reads; writer for checkout and read-your-writes (topic 17).
- **RDS Proxy** to pool connections from ~90 pods and cut failover impact.
- **Redis** cluster mode with replicas in other AZs, auto-failover; app treats cache as optional (timeouts 50 ms, DB fallback behind concurrency limits).
- Migrations: expand/contract, run as a Job before deploy (topic 35).

### 4.3 Zero-downtime deploys
- GitHub Actions builds once, signs, pushes to ECR; Argo CD + Argo Rollouts canary 5% → 25% → 100% with Prometheus analysis on 5xx and p99 (topic 34).
- Feature flags for risky features (checkout changes).

### 4.4 IaC layout (topic 36)
States: `network` (VPC, endpoints, NAT), `eks-platform` (cluster, node pools, add-ons bootstrap), `data` (Aurora, Redis, S3), `edge` (CloudFront, WAF, Route 53). Plan on PR, apply on merge via OIDC roles; policies block deletes of data resources.

---

## 5. Scaling

| Layer | How it scales | Limit to watch |
|---|---|---|
| CloudFront | Automatic | Cache hit ratio |
| ALB | Automatic (LCUs); pre-warm only for extreme sudden spikes | — |
| Pods | HPA + Karpenter; pre-scale for sales (scheduled min replicas) | vCPU quotas, subnet IPs |
| Redis | Add shards (online resharding) | Hot keys |
| Aurora | Add readers (up to 15), scale writer vertically (failover blip) | Writer CPU/connections at sale peaks |
| Async work | KEDA scales workers on queue depth | Downstream email provider limits |

Sale-day plan: load test at 1.5× forecast, pre-scale pods/readers, warm caches, freeze deploys, waiting room if needed (topic 31).

---

## 6. Reliability and DR: the 3 AM walkthrough

| What fails | What happens | User impact | Our action |
|---|---|---|---|
| One pod | Readiness fails → removed; Deployment replaces it | None | None (dashboard) |
| One node | Pods rescheduled; PDBs protect capacity | None/blip | None |
| **One AZ** | ALB stops routing there; Karpenter launches nodes in other AZs; Aurora fails over if writer was there (~30 s); Redis replicas promoted; NAT in other AZs unaffected | ~30–60 s of elevated errors for writes; reads fine | Watch; capacity was N+1 per AZ |
| Redis cluster | Cache misses → DB load ↑ (limited by concurrency limiters) | Slower pages | Page if SLO burns; scale readers |
| Aurora writer | Failover ~30 s; RDS Proxy holds connections | Checkout errors ~30 s; retries with idempotency | Verify; postmortem if longer |
| Bad deploy | Canary analysis aborts automatically | ≤ 5% of requests briefly | Team fixes forward next day |
| Payment provider down | Circuit breaker opens; secondary provider or "try again later" message | Checkout degraded | Comms; status page |
| **Region** | DR runbook: promote Aurora Global secondary, scale pilot-light EKS in DR region, flip Route 53 | Down until failover: RTO target 4 h (practised ~1.5 h) | IC declares DR (topic 41) |
| Certificate expiry | Prevented by ACM auto-renew + expiry alerts | — | — |
| Accidental data deletion | PITR restore to new cluster, repair rows | Partial | Runbook |

Availability math (topic 23): edge 99.99 × ALB 99.99 × app (3 AZ) ~99.999 × Aurora 99.99 × payment (with secondary) ~99.99 ≈ **99.95%** → meets target, with deploy safety being the biggest practical lever.

---

## 7. Security

- Edge: CloudFront + WAF (managed rules, rate rules, bot control) + Shield; origin locked to CloudFront.
- Network: private subnets for apps/data, SGs by reference, NetworkPolicies default-deny, VPC endpoints.
- Identity: SSO for humans, EKS Pod Identity per service, OIDC for CI, no static keys; SCP guardrails.
- Data: encryption at rest (KMS), TLS to Aurora/Redis, secrets via Secrets Manager + External Secrets, rotation.
- Workloads: signed images verified at admission, Pod Security restricted, runtime detection (GuardDuty EKS).
- Audit: CloudTrail org trail to log-archive account.

---

## 8. Observability

- **SLOs:** product page availability 99.95% & p99 < 500 ms; checkout success 99.95%; measured at ALB/CloudFront.
- **Alerts:** multi-window burn rates (page), synthetic checkout journey from 3 locations (page), DB/Redis saturation (ticket), cert expiry, DLQ depth.
- **Dashboards:** RED per service, USE for nodes/DB/Redis, business metrics (orders/min), deploy annotations.
- **Tracing:** OTel → Tempo/X-Ray with exemplars; logs with trace IDs.

---

## 9. Cost (rough, monthly, illustrative)

| Item | Estimate | Levers |
|---|---|---|
| EKS nodes (~30 nodes avg, autoscaled) | $6–9k | Savings Plan baseline, Spot for workers, Graviton |
| Aurora (writer + 2 readers) + storage/IO | $4–6k | Rightsize readers; I/O-Optimized if IO-heavy |
| ElastiCache | $1–2k | Reserved nodes |
| CloudFront egress (~180 TB/month) | largest variable item; negotiate/commit | Image optimization (WebP/AVIF), caching |
| NAT, endpoints, ALB, WAF | $1–2k | Endpoints reduce NAT |
| Observability | $1–3k | Retention tiers, sampling |
| DR pilot light | +15–25% | Tiered DR |

---

## 10. Trade-offs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| EKS | Team skills, portability, platform add-ons | ECS/Fargate (simpler ops) | Existing K8s expertise; ECS would be valid for a smaller team |
| Aurora | Fast failover, readers, Global DB | RDS Postgres Multi-AZ | Better RTO/RPO, small cost premium |
| Pilot-light DR | RTO 4 h at low cost | Warm standby | Business accepted 4 h |
| Canary | Small blast radius | Blue/green | High traffic → good canary statistics |

---

## 11. Follow-up questions

1. **Q: How do you get to 99.99%?**
   A: Remove remaining single-region dependency for checkout (warm standby or active-active for the API with home-region writes), automate regional failover steps, add a second payment provider with automatic failover, shrink deploy blast radius further (cells), and cut MTTR with better detection. Then confirm dependencies' SLAs allow it.

2. **Q: The DB writer is the bottleneck during sales. Options?**
   A: Short term: scale writer vertically before the sale, move more reads to readers/cache, batch writes, push non-critical writes to queues. Medium term: split hot domains into their own databases (functional partitioning), then shard orders by customer if still needed.

3. **Q: Why not just use one big EC2 instance per AZ instead of Kubernetes?**
   A: That works with an ASG and is simpler. Kubernetes helps with bin-packing many services, standardized deploys/rollouts, and platform add-ons. For a single app and small team, ASG + ALB or ECS would be the simpler, valid choice. I'd pick based on team skills and number of services.

4. **Q: What would you check in a production readiness review before launch?**
   A: SLOs and alerts, load test results at 1.5× peak, failure tests (AZ, DB failover), runbooks, backups and a tested restore, security review, cost estimate, on-call rotation, and rollback procedures ([K8s 85](../../CKA-Certification-Course-2025/notes/85-production-readiness-checklist.md)).
