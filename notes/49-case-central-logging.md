# 49 · Case Study: A Central Logging Platform

> **Level:** ★★★ advanced
>
> **You will practise:** designing a **company-wide logging platform**: volume estimation, **collection agents**, a **buffer** (Kafka) to decouple ingestion from indexing, **index-heavy vs label-index** backends (OpenSearch vs Loki), **storage tiers and retention**, **multi-tenancy and quotas**, **PII scrubbing**, **audit/security logs** with immutability, HA/DR, and **cost** (often the deciding factor).
>
> **You need:** topics 03, 18, 20–21, 25, 27, 40–41, 43.
>
> **Part 9 · Case studies** · Previous: [48 · Zero-downtime deployments](48-case-zero-downtime-deployments.md) · Next: [50 · Metrics monitoring](50-case-metrics-monitoring.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design a central logging platform for all our infrastructure and applications."

| Question | Assumed answer |
|---|---|
| Sources? | 3,000 Kubernetes nodes (~40,000 pods), 1,500 VMs, AWS service logs (CloudTrail, ALB, VPC Flow Logs), network devices |
| Volume? | ~20 TB/day raw today, growing 50%/year |
| Use cases? | (1) Debugging: search recent app logs by service/trace ID/text; (2) Security: audit logs, SIEM feed; (3) Compliance: keep audit logs 1 year+ immutable |
| Freshness? | Searchable within 60 s |
| Retention? | App logs: 14 days searchable, 90 days archive. Audit: 1 year searchable-on-demand, 7 years archive |
| Loss tolerance? | App logs: small loss acceptable in disasters; audit logs: none |
| Users? | ~800 engineers + security team; multi-tenant (teams must not see others' sensitive logs) |
| Budget? | Current SaaS bill is "too high"; aim to cut 50% |

---

## 2. Estimation

```
Ingest: 20 TB/day ÷ 10^5 s ≈ 200 MB/s avg; peak ×2.5 ≈ 500 MB/s (≈ 4 Gbps)
Lines: avg 400 B → 500 k lines/s avg, ~1.25 M lines/s peak
Growth: 50%/yr → 30 TB/day next year; design for ~2× today

Hot searchable 14 days:
  Option A, full-text index (OpenSearch): ~1.1–1.5× raw incl. 1 replica after compression → 20 TB × 14 × 1.3 ≈ 360 TB SSD
  Option B, label index + compressed chunks in object storage (Loki): compression ~10× → 20 × 14 / 10 ≈ 28 TB in S3 (+ small index)
Archive 90 days (compressed ~10×): 20 × 90 / 10 = 180 TB in S3 (cheap tiers)
Audit logs (subset ~1 TB/day): 7 years × 365 × 1 TB / 10 ≈ 255 TB in Glacier with Object Lock
```

**Conclusion:** full-text indexing everything for 14 days means hundreds of TB of SSD and dozens of nodes → expensive. Most queries are "service X, last 1 h, contains error / trace_id" → a **label-indexed store (Loki-style) on object storage** for app logs, with a **full-text index only for security/audit** data that needs it.

---

## 3. High-level design

```
SOURCES                         COLLECT                         BUFFER                 PROCESS / ROUTE                 STORE                     QUERY
K8s pods (stdout) ──▶ Fluent Bit / OTel Collector (DaemonSet) ─┐
VMs (journald/files) ─▶ Fluent Bit / Vector agent ─────────────┼─▶ Kafka (MSK)  ──▶ Vector/OTel aggregators ──┬─▶ Loki (app logs; S3 chunks) ──▶ Grafana
AWS logs (CloudTrail, ALB, VPC) ─▶ S3 → event → ingest workers ┘   topics by tier:    - parse, enrich (k8s meta) ├─▶ OpenSearch (security/audit, 30–90 d hot) ─▶ SIEM / Dashboards
Network devices ─▶ syslog collectors ──────────────────────────┘   app / audit /      - PII scrub, drop noise    └─▶ S3 archive (Parquet/compressed), Object Lock for audit ─▶ Athena (ad-hoc)
                                                                   security            - route by tenant/type
                                                                                       - quotas per tenant
```

**Flow of one log line:**
1. App writes JSON to stdout → container runtime file → Fluent Bit tails it, adds K8s metadata (namespace, pod, labels), buffers to disk if downstream is slow.
2. Fluent Bit sends to Kafka topic `logs-app` (partitioned by namespace) with `acks=all`.
3. Aggregators consume, parse, scrub PII patterns (cards, emails, tokens), drop/sample noisy lines (health checks, debug), apply per-tenant rate limits, then write:
   - app logs → Loki (labels: cluster, namespace, app, level; content stays unindexed, compressed in S3);
   - audit/security → OpenSearch + S3 archive with Object Lock.
4. Engineer queries in Grafana: `{namespace="payments", app="api"} |= "trace_id=4bf92f..."` → Loki finds streams by label, scans compressed chunks in parallel.

---

## 4. Deep dives

### 4.1 Why a buffer (Kafka)?
- Backends slow down (compactions, upgrades, query storms). Without a buffer, agents back up and **drop logs** or consume node disk.
- Kafka absorbs hours of backlog (retention 24–72 h), lets multiple consumers read the same stream (Loki, SIEM, archive), and allows **replay** after a backend outage.
- Size (topic 21): storage follows the **average** rate (200 MB/s), throughput the **peak** (500 MB/s). 48 h retention with RF 3 → 200 × 3 × 172,800 s ≈ 104 TB of broker disk: too much. Use **tiered storage** or keep local retention 12 h (200 × 3 × 43,200 ≈ 26 TB) and replay older data from the S3 archive. Brokers must sustain 500 MB/s in + 1 GB/s replication + consumer reads at peak. Partitions: ~10 MB/s each → 50–100 partitions per topic.
- Alternative at smaller scale: agents → Loki directly with agent disk buffers, or Kinesis Firehose → S3.

### 4.2 Backend choice

| | OpenSearch/Elasticsearch | Loki (label index) | Object store + SQL (Athena/ClickHouse) |
|---|---|---|---|
| Query style | Full-text search, aggregations, fast | Filter by labels, then grep content | SQL over columnar files |
| Cost at 20 TB/day | Very high (SSD, many nodes) | Low (S3) | Low storage, pay per scan / cluster |
| Ingest complexity | Mappings, shard management, hot/warm tiers | Simple; label cardinality must be controlled | Batch-oriented (minutes) |
| Best for | Security analytics, dashboards over fields | Debugging app logs | Long-term investigation, compliance queries |

Decision: **Loki for app logs**, **OpenSearch (smaller, ~1–2 TB/day) for security/audit** feeding the SIEM, **S3 + Athena** for archive queries.

### 4.3 Labels and cardinality (Loki)
Labels: `cluster, namespace, app, container, level` (bounded). **Never** `trace_id`, `user_id`, `pod` (high churn), request path → search those as content filters or structured metadata. Too many streams = memory blow-ups in ingesters.

### 4.4 Multi-tenancy and quotas
- Tenant = team (mapped from namespace → team via service catalog).
- Per-tenant ingestion rate limits and retention overrides; per-tenant query limits (max time range, concurrency).
- Access: Grafana teams/folders + Loki tenant headers via an auth proxy; security logs only for the security team.
- **Showback:** GB/day per team dashboard → teams reduce their own noise.

### 4.5 PII and secrets
- App guidelines (structured logging with allow-listed fields) + pipeline scrubbing (regex for card numbers/tokens, hashing user identifiers).
- Detection: scanner on samples; alerts when PII patterns appear; deletion workflow (Loki/OpenSearch delete APIs; S3 lifecycle) for GDPR requests.

---

## 5. Scaling
- Agents: one per node, small CPU/memory limits, disk buffer.
- Kafka: add brokers/partitions as volume grows.
- Aggregators: stateless, HPA on consumer lag.
- Loki: horizontally scalable microservices mode (distributors, ingesters, queriers, compactor); ingesters scale with stream count; queriers scale with query load; S3 scales automatically.
- OpenSearch: hot (SSD) → warm (UltraWarm/S3-backed) tiers, ISM policies for rollover by size/age; shard size target 10–50 GB.

---

## 6. Reliability and DR
| Failure | Effect | Mitigation |
|---|---|---|
| Loki ingester crash | In-memory data at risk | WAL on ingesters + replication factor 3 across AZs |
| Loki/OpenSearch down | No search; ingestion continues into Kafka | Kafka buffer; replay when back |
| Kafka down | Agents buffer to local disk (minutes–hours) | Multi-AZ MSK; agent disk buffers sized for ~1 h |
| Region loss | Logging for that region gone | App logs: accept (degrade); audit logs: S3 CRR to another region/account (Object Lock) |
| Log storm (one app logs 50× normal) | Pipeline lag for everyone | Per-tenant rate limits at aggregator; drop with a counter metric |

SLOs: ingestion freshness (99% of lines searchable within 60 s), query availability 99.5%, audit log durability (no loss: end-to-end acks, delivery counters reconciled).

---

## 7. Security
- TLS from agents to Kafka (mTLS), encryption at rest (KMS) everywhere.
- Audit logs: separate account, S3 Object Lock (compliance mode), restricted access, integrity validation (CloudTrail digest files).
- RBAC per tenant; query audit (who searched what).
- Logs are sensitive: treat the platform as a high-value target.

---

## 8. Observability of the platform
- Agent: dropped records, buffer usage, retries.
- Kafka: consumer lag per group, under-replicated partitions, disk.
- Loki: ingestion rate per tenant, rejected samples (rate limits), stream count, query latency, compactor health.
- End-to-end **canary log line**: a synthetic job logs a unique line every minute in every cluster; an alert fires if it's not searchable within 2 minutes.

---

## 9. Cost (rough, illustrative)
```
Old SaaS at ~$0.10–0.30/GB ingested × 600 TB/month = $60k–180k/month
New:
  Loki: S3 ~30–40 TB hot + compute (ingesters/queriers) ~ $10–20k/month
  OpenSearch for 1–2 TB/day security logs: ~ $10–15k/month
  Kafka (MSK + tiered storage): ~ $8–12k/month
  Archive S3 IA/Glacier: ~ $2–4k/month
  Engineering: 2–3 FTE platform team (the hidden cost!)
Volume reduction (drop noise, sample debug, shorter retention for chatty services): often −30–50% raw
```

---

## 10. Trade-offs

| Decision | Chosen | Gave up |
|---|---|---|
| Loki for app logs | ~5–10× cheaper storage, simple ops | Fast full-text search across everything; must filter by labels first |
| Kafka buffer | No data loss on backend issues, replay, fan-out | Extra system and cost |
| Self-hosted vs SaaS | Cost at 20 TB/day | Ops burden; SaaS features |
| Pipeline scrubbing | Defence in depth for PII | CPU cost, imperfect regex coverage |

---

## 11. Follow-up questions

1. **Q: Engineers complain searches across 14 days are slow. Options?**
   A: Encourage narrower label selectors and time ranges; add structured metadata/bloom filters for high-cardinality fields like trace IDs (Loki supports this); keep recent data in faster caches; derive metrics from logs (recording rules) so dashboards don't scan logs; for frequent full-text use cases, route that subset to OpenSearch.

2. **Q: How do you guarantee audit logs are never lost?**
   A: End-to-end acknowledgements (agent disk buffer → Kafka `acks=all` with min ISR 2 → idempotent writers to S3), separate pipeline from app logs, delivery counts reconciled between source (CloudTrail digest, counters) and archive, Object Lock and cross-region replication.

3. **Q: One team's debug logging floods the platform at 3 AM. What happens?**
   A: Their tenant hits the ingestion rate limit; excess is dropped (with a metric and a notification to the team), other tenants are unaffected because limits are per tenant and Kafka absorbs short bursts. The team gets a ticket and showback shows the cost.

4. **Q: Why not just use CloudWatch Logs for everything?**
   A: For small or AWS-only setups it's a fine choice. At 20 TB/day, ingestion pricing makes it very expensive, cross-account/cluster querying is limited, and multi-cloud/on-prem sources need a separate solution. A hybrid is common: CloudWatch for AWS service logs with short retention, exported to the central platform.
