# 51 · Case Study: Multi-Region Kubernetes

> **Level:** ★★★ advanced
>
> **You will practise:** designing a **Kubernetes platform across regions**: cluster-per-region topology, **fleet management with GitOps** (ApplicationSets, cluster bootstrapping), **global traffic management**, handling **stateful workloads and data** across regions, **regional independence**, **progressive rollouts across the fleet**, **failover and failback**, observability and **cost**.
>
> **You need:** topics 07–09, 17, 19, 23, 33–34, 36–38, 41–42.
>
> **Part 9 · Case studies** · Previous: [50 · Metrics monitoring](50-case-metrics-monitoring.md) · Next: [52 · Database DR](52-case-database-dr.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Our SaaS runs on one EKS cluster in Mumbai. Design a multi-region Kubernetes setup so a region outage doesn't take us down."

| Question | Assumed answer |
|---|---|
| Target RTO/RPO for region loss? | RTO 15 min, RPO < 1 min for core data |
| Active-active or passive? | Business wants active-active for latency to SEA users too |
| Regions? | `ap-south-1` (Mumbai), `ap-southeast-1` (Singapore); later EU |
| Workloads? | 70 stateless services, Aurora PostgreSQL, Redis, Kafka (MSK), S3, OpenSearch |
| Data residency? | Indian customers' PII must stay in India? → Assume: allowed in Singapore for DR, but primary in India |
| Traffic? | 8,000 req/s peak; 70% India, 30% SEA |
| Team? | Platform team of 8; already uses Terraform + Argo CD |

---

## 2. Estimation

```
Each region must handle 100% of traffic during failover → each sized for 8,000 req/s at N+1 per AZ
Normal: Mumbai 70% (5,600 req/s), Singapore 30% (2,400 req/s) → Singapore runs at ~30% of its failover capacity
   → keep Singapore pre-scaled to ~60% of full (warm) and rely on autoscaling for the rest (+quotas raised),
     or full capacity in both (≈2× compute). Choose: ~60% warm + Karpenter (validated in game days).
Data replication Mumbai → Singapore: write rate ~5 MB/s DB WAL + S3 new objects 2 TB/day
   cross-region transfer ≈ (5 MB/s × 2.6 M s ≈ 13 TB) + 60 TB S3/month → ~73 TB × $0.02/GB (illustrative) ≈ $1.5k/month
```

---

## 3. High-level design

```
                     Global Accelerator (anycast IPs) / Route 53 latency + health checks
                       │                                           │
             ┌─────────▼──────────┐                       ┌────────▼───────────┐
             │ ap-south-1 (home   │                       │ ap-southeast-1      │
             │ for India tenants) │                       │ (home for SEA)      │
             │ EKS prod-aps1      │                       │ EKS prod-apse1      │
             │  70 services       │                       │  70 services        │
             │ Aurora Global DB   │══ async (<1 s) ═══════│ Aurora secondary    │ (promotable)
             │  PRIMARY           │                       │  (reads local)      │
             │ Redis (regional)   │                       │ Redis (regional)    │
             │ MSK (regional)     │── MirrorMaker 2 ──────│ MSK                 │ (only replicated topics)
             │ S3 ════ CRR ═══════════════════════════════│ S3                  │
             └────────────────────┘                       └─────────────────────┘
Management: Argo CD per cluster ← GitOps repo (ApplicationSets over cluster list) ; Terraform per region
Observability per region + global view ; CI builds once → ECR replicated to both regions
```

**Write model:** single global primary for the core relational data (Aurora Global DB primary in Mumbai). Singapore serves **reads locally** and sends **writes to Mumbai** (cross-region latency ~60 ms RTT for writes from SEA users: acceptable). On region failure, promote the Singapore secondary.
(Alternative: home-region per tenant with separate primaries, more complex, chosen later for EU expansion with residency.)

**Request flow (SEA user, read):** anycast → Singapore ALB → pod → local Redis/Aurora reader → response (~20 ms).
**Request flow (SEA user, write):** Singapore pod → Aurora **writer endpoint in Mumbai** (via Global DB write forwarding or a direct connection over inter-region peering/TGW) → commit → replicated back within ~1 s; app uses read-your-writes by reading from the writer for that user briefly (topic 17).

---

## 4. Deep dives

### 4.1 Fleet management (GitOps)
```
gitops/
├─ clusters/prod-aps1/ (cluster values: region, capacity, endpoints)
├─ clusters/prod-apse1/
├─ platform/ (add-ons bundle: ingress, cert-manager, ESO, Karpenter, monitoring) — versioned
└─ apps/<service>/ (chart/values) + envs/prod/<region>/<service>.yaml (digest)
Argo CD in each cluster (no single management cluster as SPOF for runtime);
ApplicationSet with a cluster generator → one Application per service per cluster
```
- **Bootstrap:** Terraform creates cluster + installs Argo CD + points it at `clusters/<name>` → everything else syncs. A new region = new Terraform stack + a new folder.
- **Region-specific config:** endpoints (local Redis/Kafka/readers), capacity, feature flags.

### 4.2 Progressive rollouts across regions
Region waves (topic 34): canary in Singapore first (smaller traffic, lower risk) → bake 1 h → Mumbai. Platform bundle upgrades: dev → staging → Singapore → Mumbai.

### 4.3 Stateful components
| Component | Strategy |
|---|---|
| Aurora PostgreSQL | Global Database: primary Mumbai, secondary Singapore (RPO ~1 s, managed failover in minutes) |
| Redis | Regional, independent, cache only; cold in DR region is OK (warm-up) |
| Kafka | Regional clusters; **MirrorMaker 2** replicates topics needed for DR; consumers must handle offsets translation and idempotency (topic 22) |
| S3 | Cross-Region Replication (with Replication Time Control for 15-min SLA on most objects) |
| OpenSearch | Rebuildable index from DB/events; cross-cluster replication if rebuild too slow for RTO |
| Secrets | Multi-region secrets; multi-region KMS keys |
| In-cluster PVs | Avoid for important data; if needed, application-level replication |

### 4.4 Regional independence check
- Auth: JWT validation local in each region (JWKS cached); the IdP is SaaS (multi-region).
- No service in Singapore calls a Mumbai service synchronously, except **DB writes** (accepted, explicit). During a Mumbai outage, writes fail until promotion → documented.
- CI/CD: deploys to Singapore don't depend on Mumbai (ECR replication, Argo CD local).
- Observability: each region has its own stack; global dashboards federate.

### 4.5 Failover runbook (Mumbai lost)
```
0. Detect: multi-signal (health checks from 3 external locations, AWS Health, SLO burn) → IC decision ≤ 5 min
1. Traffic: Global Accelerator/Route 53 send all users to Singapore (automatic on health checks for reads)
2. Data: Aurora Global DB failover (unplanned: "detach and promote" Singapore) ~2–5 min ; RPO ~1 s
3. Config flip: apps in Singapore point writes to local writer (feature flag / config change via GitOps or runtime config)
4. Capacity: raise HPA minimums, Karpenter scales nodes (quotas pre-raised)
5. Kafka: consumers in Singapore switch to mirrored topics; idempotent processing handles duplicates
6. Verify SLIs; comms
Failback: rebuild Mumbai as secondary, sync, planned switchover in low traffic
```
Target total ≈ 10–15 min. Practised quarterly (topic 44).

---

## 5. Scaling
- Add regions by adding a Terraform stack + GitOps cluster folder; with the single-primary model, write latency grows for far regions → EU expansion triggers the **home-region per tenant** model (topic 42).
- Per-region Karpenter, quotas, and IP planning (non-overlapping CIDRs for inter-region connectivity).

---

## 6. Reliability and DR summary
| Failure | Result |
|---|---|
| Pod/node/AZ | Handled inside region (topic 46) |
| Singapore region | Anycast routes everyone to Mumbai; no data promotion needed; Mumbai sized for 100% |
| Mumbai region | Runbook above: RTO ~15 min, RPO ~1 s |
| Bad global config/deploy | Region waves limit to one region |
| Argo CD down in one region | Running apps unaffected; deploys paused there |

---

## 7. Security
- Same baseline in both regions via the platform bundle (policies, Pod Security, signed images).
- Inter-region traffic encrypted (TLS to Aurora writer, TGW inter-region peering encrypted by AWS).
- IAM roles per region; SCPs allow only the two regions.
- Data residency documented: Singapore holds replicas of Indian data for DR (approved).

---

## 8. Observability
- Per-region stacks + global view (topic 50); SLOs per region and global.
- Replication lag alerts: Aurora Global DB lag > 5 s, S3 replication backlog, MirrorMaker lag.
- Synthetic checks per region and through the global entry point.
- DR readiness dashboard: quotas headroom, image replication status, secret replication, last successful game day.

---

## 9. Cost
- Compute: Singapore at ~60% warm of full capacity → ~1.6–1.8× single-region compute in total.
- Aurora Global DB: secondary cluster instances + replicated write I/O.
- Cross-region data transfer (~$1–2k/month estimate above), MSK duplicate cluster, observability ×2.
- Total likely +70–100% over single region → justified by RTO 15 min and SEA latency.

---

## 10. Trade-offs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Single global DB primary | Simple consistency | Home region per tenant / multi-writer | Lower complexity now; revisit for EU |
| Argo CD per cluster | No cross-region dependency | Central Argo CD hub | Blast radius and independence |
| Warm (60%) secondary | Cost | Full capacity | Validated autoscaling in drills |
| Anycast (Global Accelerator) | Fast failover, static IPs | DNS only | Clients caching DNS would slow failover |

---

## 11. Follow-up questions

1. **Q: Why not stretch one EKS cluster across two regions?**
   A: The control plane (etcd quorum) needs low latency and is regional; a stretched cluster makes a region failure a cluster failure and adds cross-region traffic for everything. Cluster per region with GitOps gives independence.

2. **Q: How do you make sure the second region actually works?**
   A: It serves real traffic every day (active-active for reads and SEA users), every deploy goes there first, DR readiness checks run continuously, and we run quarterly failover drills that measure RTO/RPO.

3. **Q: What if replication lag is 30 s when Mumbai dies?**
   A: We lose up to 30 s of writes (RPO breach). Mitigations: alert on lag early, throttle heavy batch writers, the outbox/event log lets us reconcile and replay some lost actions, and customers are notified per the incident process. If RPO must be ~0 cross-region, we'd need a consensus-based global database and accept higher write latency.

4. **Q: How would you add an EU region with residency requirements?**
   A: Move to home-region-per-tenant: EU tenants' primary data in the EU (with EU-only DR pair), a routing layer mapping tenant → region, region-local observability/logs, and no replication of EU personal data outside the EU.
