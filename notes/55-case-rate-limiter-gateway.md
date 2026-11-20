# 55 · Case Study: A Distributed Rate Limiter / API Gateway

> **Level:** ★★★ advanced
>
> **You will practise:** designing a **rate-limiting service** behind an **API gateway** for a public API: requirements (limits per key, per plan, per endpoint), algorithm choice, **where the counter lives** (local vs Redis vs hybrid), **atomicity**, **latency budget**, **multi-region** limits, **fail open vs closed**, **hot keys**, configuration delivery, observability and cost.
>
> **You need:** topics 10, 14, 18–19, 24–26.
>
> **Part 9 · Case studies** · Previous: [54 · Secrets management](54-case-secrets-management.md) · Next: [56 · Job scheduler](56-case-job-scheduler.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design rate limiting for our public API platform."

| Question | Assumed answer |
|---|---|
| Who is limited? | API keys (per customer), with per-plan limits; anonymous traffic per IP; some per-endpoint limits |
| Limits? | Free 10 req/s (burst 20), Pro 200 req/s (burst 400), Enterprise custom (up to 5,000 req/s); daily quotas per plan |
| Traffic? | 150,000 req/s peak total across 2 regions; ~200,000 active keys/day |
| Latency budget? | Limiter adds < 2 ms p99 |
| Accuracy? | Within ~5–10% is OK for rate limits; daily quotas must be accurate for billing (±0.1%) |
| Failure? | Limiter failure must not take the API down |
| Config changes? | Plans updated by sales/admin; must apply within 1 minute |
| Response? | 429 + headers (limit, remaining, reset) |

---

## 2. Estimation

```
Checks: 150,000 req/s → 150,000 limiter decisions/s (+ endpoint-specific checks ~×1.3 → ~200,000/s)
Redis: one atomic Lua call per decision; a Redis shard handles ~100k simple ops/s, Lua scripts fewer (~50–80k/s)
   → ~4–6 shards per region with headroom (plus replicas)
State: 200,000 keys × (bucket hash ~100 B) = 20 MB → memory is trivial; ops/s is the constraint
Network: 200k/s × ~200 B = 40 MB/s to Redis per region → fine
Quotas (daily counts): aggregate locally, flush every few seconds → 150k/s becomes ~1–2k writes/s to the counter store
```
**Conclusion:** the challenge is **ops/s and latency**, not storage; plus resilience and accuracy trade-offs.

---

## 3. High-level design

```
Client ─▶ CDN/WAF (per-IP coarse limits, bot rules)
        ─▶ NLB ─▶ API Gateway fleet (Envoy/Kong), per region, ~40 pods
                    │ 1. auth: API key → plan (local cache, refreshed via config stream)
                    │ 2. local token bucket pre-check (absorbs abusive keys cheaply)
                    │ 3. global check: Redis Cluster (Lua token bucket), timeout 3 ms
                    │ 4. allow → route to backend ; deny → 429 + headers
                    ▼
            Redis Cluster (regional, 6 shards × primary+replica across AZs)
Quota pipeline: gateway pods aggregate per-key counts in memory → every 5 s → Kafka → quota service → DB (billing-accurate)
Config: plans/limits in a config DB → change events → gateways (push, cached, versioned)
Observability: decisions/s, 429s by plan/key, Redis latency, fail-open events
```

**Flow of one request:**
1. Gateway validates API key (cached lookup: key hash → customer, plan).
2. Local bucket for this key on this pod (generous: e.g. 2× plan rate / pod count) → rejects floods early without network.
3. Redis Lua: refill + take a token atomically for `rl:{key}` (hash tag keeps a key's data in one slot) → returns allowed, remaining, reset.
4. If Redis times out (> 3 ms) → **fail open** using the local bucket at the per-pod share and emit a metric.
5. Allowed → forward to backend; usage counter incremented in memory for quotas.

---

## 4. Deep dives

### 4.1 Algorithm
**Token bucket** (rate + burst) for per-second limits (topic 25); **fixed daily window counters** for quotas (billing-friendly: "1 M calls/day"). Sliding window counter for specific endpoints where edge bursts matter (login).

### 4.2 Where the counter lives
| Option | Accuracy | Latency | Resilience |
|---|---|---|---|
| Local only (limit ÷ pods) | Poor when traffic is uneven across pods (LB stickiness, few connections per big client) | ~0 | Excellent |
| **Central Redis per request** ✅ (with local pre-check) | Good | +0.5–1 ms | Depends on Redis (mitigated by fail-open) |
| Hybrid: local buckets synced every 100 ms | Approximate (overshoot up to sync interval × rate) | ~0 | Good |

Big enterprise keys (5,000 req/s) generate heavy load on one Redis key → for keys above a threshold, use **hybrid mode** (local buckets with periodic sync/leases: each pod leases a chunk of tokens, e.g. 100 at a time, from Redis), cutting Redis ops by ~100×.

### 4.3 Atomicity and correctness
- Lua script executes atomically on the shard (no race between read and write).
- Use Redis server time (`TIME`) inside the script, or pass a monotonic-ish timestamp; avoid client clock skew affecting refill (topic 26).
- Keys expire after inactivity (TTL) → memory stays bounded.

### 4.4 Multi-region
- Exact global limits would need a cross-region call per request (~50–200 ms) → unacceptable.
- **Per-region budgets:** each region enforces a share of the key's limit (e.g. by recent traffic split, adjusted every minute by a controller), or **home-region enforcement** where clients mostly hit one region.
- Quotas (billing) are aggregated globally asynchronously; slight overshoot possible within the aggregation delay (seconds) → acceptable and documented; hard stop when the global count passes 100% + small grace.

### 4.5 Fail open vs fail closed
- General API limits: **fail open** with local per-pod limits (availability first, protection still roughly in place).
- Security-sensitive endpoints (login, OTP): **fail closed** (or degrade to strict local limits).
- Alert when fail-open rate > 0.1% of decisions.

### 4.6 Config delivery
Plan changes → DB → change event → gateways update in-memory cache; full refresh every 5 minutes as safety; versioned config; invalid config rejected (schema validation) and gateways keep last good config (static stability).

---

## 5. Scaling
- Gateways: stateless, HPA on CPU/requests.
- Redis: add shards (resharding online); hot keys handled by leasing.
- Quota pipeline: Kafka partitions by key; quota service scales by partitions.

---

## 6. Reliability and DR
| Failure | Effect | Mitigation |
|---|---|---|
| Redis shard failover (~10–30 s) | Keys on that shard fail open locally | Local buckets keep rough protection |
| Whole Redis cluster down | All keys fail open locally | Alert; scale gateway local limits conservatively |
| Config service down | Gateways keep last known plans | Static stability |
| Region down | Traffic shifts to other region; per-region budgets rebalance (controller) or temporarily allow full limit in surviving region | Runbook |
| Abusive key floods | Local pre-check rejects before Redis | Also WAF/IP rules |

---

## 7. Security
- API keys stored hashed; key rotation without downtime (two active keys per customer).
- Limits per IP for unauthenticated endpoints; credential stuffing protections on login (fail closed).
- Prevent key enumeration (constant-time comparisons, generic errors).
- Redis in private subnets, TLS + AUTH/ACLs.

---

## 8. Observability
- Metrics: decisions/s, allow/deny ratio by plan, top limited keys, Redis p99 latency, Lua errors, fail-open count, local vs global decision counts, quota pipeline lag.
- Customer-facing: usage dashboards and headers so customers can self-diagnose 429s.
- Alerts: Redis latency > 2 ms p99 (ticket), fail-open spike (page), quota pipeline lag > 5 min (ticket; billing accuracy).

---

## 9. Cost
- Redis: 6 shards × 2 (replicas) × 2 regions of mid-size nodes → modest.
- Gateway CPU overhead for limiter logic: small.
- Main trade-off is engineering complexity of hybrid/multi-region logic vs accuracy needs.

---

## 10. Trade-offs

| Decision | Chosen | Gave up |
|---|---|---|
| Redis per request + local pre-check | Good accuracy, cheap rejection of floods | ~1 ms latency, Redis dependency |
| Token leasing for huge keys | Redis load ÷ 100 | Small overshoot |
| Per-region budgets | No cross-region latency | Exact global limits |
| Fail open (general) | API availability | Brief loss of precise limiting |

---

## 11. Follow-up questions

1. **Q: A customer says they're getting 429s below their limit. Why might that be?**
   A: Per-region budget split doesn't match their traffic split, local per-pod pre-check is too strict for uneven pod distribution (few long-lived connections pinned to few pods), burst exceeds bucket capacity, or endpoint-specific limits apply. Check decision logs for their key and which layer denied.

2. **Q: How would you rate limit by "cost" instead of request count (e.g. expensive search)?**
   A: Weighted tokens: each endpoint has a cost; the Lua script takes N tokens. Or measure actual backend cost (CPU-ms) and debit asynchronously, with a concurrency limit for expensive endpoints.

3. **Q: Why not just use the cloud API gateway's built-in throttling?**
   A: For many cases it's enough (usage plans with rate/burst per key). At 150k req/s, per-request pricing and limited customization (per-endpoint weights, multi-region budgets, custom headers, fail-open policies) push us to a self-hosted gateway with our own limiter.

4. **Q: How do you test the limiter?**
   A: Unit tests for the algorithm (time-controlled), load tests with many keys and hot keys, chaos tests (Redis failover → verify fail-open behaviour and latency), and shadow mode in production before enforcing new limits.
