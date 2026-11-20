# 50 · Case Study: A Metrics Monitoring and Alerting Platform

> **Level:** ★★★ advanced
>
> **You will practise:** designing a **Prometheus-based metrics platform** for many clusters: series and sample-rate estimation, **per-cluster collection** with **remote write** to a **horizontally scalable long-term store** (Mimir/Thanos/Cortex or managed Prometheus), **HA Prometheus pairs** and deduplication, **cardinality control** and tenant limits, **recording rules**, **alerting architecture** (rulers, Alertmanager HA, routing), **meta-monitoring**, **retention and downsampling**, and **cost**.
>
> **You need:** topics 03, 18, 21, 26–29, 43.
>
> **Part 9 · Case studies** · Previous: [49 · Central logging](49-case-central-logging.md) · Next: [51 · Multi-region Kubernetes](51-case-multi-region-kubernetes.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design monitoring and alerting for 40 Kubernetes clusters across 3 regions."

| Question | Assumed answer |
|---|---|
| Scale? | 40 clusters, ~6,000 nodes, ~80,000 pods |
| What's monitored? | K8s (nodes, pods, control plane), apps (RED via OTel/Prometheus client), databases, cloud services, synthetic checks |
| Query needs? | Dashboards per service and fleet-wide; queries across clusters ("error rate of service X in all regions") |
| Retention? | 15 days raw; 13 months downsampled (capacity planning, yearly comparisons) |
| Alerting? | SLO burn-rate alerts per service, routed to ~60 teams via PagerDuty/Slack |
| Availability? | Alerting pipeline 99.9%+; must alert even if one region is down |
| Constraints | Teams sometimes add high-cardinality labels; the current single Prometheus per cluster keeps OOMing |

---

## 2. Estimation

```
Series per pod (app + cAdvisor + kube-state-metrics share): ~400 → 80,000 × 400 = 32 M
Series per node (node-exporter, kubelet): ~1,500 → 6,000 × 1,500 = 9 M
Other (DBs, cloud exporters, control plane, blackbox): ~4 M
Total active series ≈ 45 M (and churn: pods restart → new series; plan for ~1.5× in the head)

Scrape interval 30 s → samples/s = 45 M / 30 = 1.5 M samples/s
Storage: ~1.5 bytes/sample compressed → 1.5 M × 86,400 × 1.5 B ≈ 195 GB/day → 15 days ≈ 3 TB (× replication)
13 months downsampled (5-min resolution, ~10% of raw size) ≈ 195 GB × 0.1 × 395 ≈ 7.7 TB
Per cluster: ~1.1 M series avg, large clusters 5 M+ → a single Prometheus per big cluster is at its limit (memory ~ few KB/series → 15–30 GB+ RAM)
```
**Conclusion:** keep **collection local** to each cluster (lightweight), but move **storage, long-term retention and global querying** to a **horizontally scalable backend**; control cardinality.

---

## 3. High-level design

```
Each cluster (×40)
  Prometheus in AGENT mode (or OTel Collector) ×2 (HA pair, scrape same targets)  ── remote_write (external_labels: cluster, region, replica)
  kube-state-metrics, node-exporter, app /metrics, blackbox exporter
           │
           ▼  (per region, private connectivity)
  ┌──────────── Regional metrics backend: Grafana Mimir (or Thanos / Cortex / AMP) ────────────┐
  │ distributors (validate, per-tenant limits, HA dedup by replica label)                         │
  │ ingesters (RF 3 across AZs, WAL) → blocks to S3                                              │
  │ store-gateways + queriers + query-frontend (caching, splitting) ← Grafana                    │
  │ compactor (dedupe, downsample)   ruler (recording + alerting rules)                          │
  └───────────────────────────────────────────────────────────────────────────────────────────────┘
           │ alerts                                         ▲ global queries (federated query across regions)
           ▼                                                │
  Alertmanager cluster (3 replicas across AZs/regions, gossip) → routing by team label → PagerDuty / Slack / tickets
  Meta-monitoring: separate small Prometheus + external dead-man's switch (heartbeat to PagerDuty)
```

**Flow:** pod exposes `/metrics` → in-cluster agent scrapes every 30 s, adds `cluster`/`region` labels → remote-writes to the regional Mimir (two agents send identical samples; distributor deduplicates by HA replica label) → ingesters hold recent data in memory + WAL, flush 2-hour blocks to S3 → queries hit ingesters (recent) + store gateways (S3) → ruler evaluates SLO rules and sends alerts to Alertmanager → routed by `team` label.

---

## 4. Deep dives

### 4.1 Why agent mode + remote write (vs full Prometheus per cluster + federation)?
| Approach | Pros | Cons |
|---|---|---|
| Full Prometheus per cluster + Thanos sidecar | Local querying and alerting survive backend outage | Each cluster runs a stateful TSDB; global queries fan out to 40 clusters |
| **Agent + remote write to central backend** ✅ | Lightweight clusters, one place for long-term storage, global queries fast | Backend is a critical dependency; alerting must be resilient to backend issues |
| Hierarchical federation | Simple | Doesn't scale, loses detail |
| Managed (AMP/GCP Managed Prometheus/Azure Monitor) | No ops | Cost per sample/series; limits |

Mitigation for backend dependency: **regional** backends (a region outage only affects that region), plus each cluster keeps a few **critical local alerts** (e.g. via a small local Prometheus/vmalert for "cluster API down") as a fallback.

### 4.2 Cardinality control
- Per-tenant limits: max series per tenant, max label names, max label value length, samples/s; requests over the limit rejected with clear errors.
- Scrape-time relabelling to drop unused metrics (often 30–50% of default exporter output is unused).
- CI check for new metrics (lint: no `user_id`, `path` with IDs) and a weekly "top series" report per team.
- Bounded labels: `route` templates instead of raw paths, `status_class` (2xx/5xx) where detail isn't needed.

### 4.3 Recording rules and SLOs
- Precompute expensive expressions (e.g. per-service error ratios over 5m/1h/6h/3d) → fast dashboards and alerts.
- SLO definitions as code (Sloth/Pyrra) → generated recording and burn-rate rules per service (topics 28–29).

### 4.4 Alerting architecture
- **Rulers** evaluate alerts in the backend (fleet-wide rules) + **local rules** for cluster-critical alerts.
- **Alertmanager** HA (3+ replicas, gossip; dedupes notifications); routing tree by `team`/`severity`; inhibition (cluster-down inhibits pod alerts); silences with expiry.
- **Routing from catalog:** `team` label injected from the service catalog (namespace → team) so alerts reach the right owner.
- **Dead man's switch:** `Watchdog` alert always firing → external heartbeat service pages if it stops.

---

## 5. Scaling
- Distributors/queriers stateless → HPA.
- Ingesters scale with active series (~1–2 M series per ingester depending on size) → 45 M × RF 3 ≈ 135 M series held → ~70–100 ingesters across regions; shuffle sharding per tenant to limit noisy-tenant blast radius.
- Query frontend: split long queries by day, cache results (memcached), limit max range per query.
- S3: no limit; compactor scales by tenants/blocks.

---

## 6. Reliability and DR
| Failure | Effect | Mitigation |
|---|---|---|
| One Prometheus agent dies | No gap (HA pair) | Dedup in distributor |
| Network to backend broken | Agents buffer (WAL, ~2 h) | Retries; alert on remote-write lag |
| Ingester loss | No data loss | RF 3 + WAL |
| Regional backend down | That region's dashboards/global rules down | Local critical alerts in clusters; other regions unaffected; synthetic checks from outside |
| Alertmanager down | No notifications | HA replicas across zones/regions + dead man's switch from external |
| Cardinality explosion from a deploy | Tenant hits limits; others protected | Limits per tenant + alert to owner |

**SLOs for the platform:** alert delivery latency (p99 < 2 min from condition true), ingestion success 99.9%, query availability 99.5%.

---

## 7. Security
- mTLS/auth from agents to backend; tenant ID from authenticated identity, not from a header the client sets freely.
- RBAC in Grafana per team; metrics can contain sensitive info (hostnames, customer names in labels → prohibited).
- Encrypted S3 buckets; least-privilege IAM for compactor/store-gateways.

---

## 8. Observability of the platform (meta-monitoring)
- Separate small Prometheus per region scraping the backend components; dashboards: ingestion rate, rejected samples by reason, active series per tenant, ingester memory, query latency, ruler evaluation failures, Alertmanager notification failures.
- External synthetic: push a known metric and check it can be queried; heartbeat alert.

---

## 9. Cost (rough)
```
Self-hosted Mimir: S3 (~5 TB raw ×? + downsampled ~8 TB) small; compute for ingesters (memory-heavy) dominates
  → maybe $15–30k/month for 45 M series (illustrative)
Managed Prometheus priced per samples ingested: 1.5 M samples/s ≈ 3.9 T samples/month → at ~$0.90 per 10 M samples (illustrative tier) ≈ $350k/month
  → cardinality and scrape interval decide cost: 60 s scrape halves samples; dropping unused series cuts 30–50%
Levers: drop unused metrics, longer scrape intervals for non-critical targets, downsampling, per-team showback of series
```

---

## 10. Trade-offs

| Decision | Chosen | Gave up |
|---|---|---|
| Agent + remote write | Light clusters, central long-term store | Local querying during backend outages (mitigated by local critical alerts) |
| Regional backends | Blast radius per region | Global queries need federation across 3 backends |
| Self-hosted Mimir | Cost at 45 M series | Ops burden (dedicated team) |
| 30 s scrape | Good resolution | 2× cost vs 60 s |

---

## 11. Follow-up questions

1. **Q: A team adds `user_id` to a metric and series jump by 20 M. What happens and what do you do?**
   A: Their tenant hits its series limit; new series are rejected (their metric breaks, other tenants are unaffected). They're alerted with the offending metric name. Fix: remove the label (that detail belongs in logs/traces) and add a CI lint rule.

2. **Q: How do you make sure alerts still fire if the metrics backend is down?**
   A: Critical alerts evaluated close to the data (local rules per cluster), external synthetic checks independent of the backend, Alertmanager HA, and a dead man's switch so silence itself pages someone.

3. **Q: Thanos vs Mimir?**
   A: Both give long-term storage on object storage and global querying. Thanos fits a sidecar model (keep Prometheus per cluster, upload blocks, query fan-out). Mimir is a push-based, multi-tenant, horizontally scalable service with strong per-tenant limits. For many clusters with remote write and multi-tenancy, Mimir-style is simpler to operate centrally; Thanos suits keeping autonomous Prometheus servers.

4. **Q: How do you keep monitoring cost under control as clusters grow?**
   A: Per-tenant series limits and showback, dropping unused metrics at scrape time, recording rules instead of raw high-resolution queries, downsampling for long retention, scrape intervals by importance, and reviewing top-cardinality metrics monthly.
