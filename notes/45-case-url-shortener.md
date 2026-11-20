# 45 · Case Study: A URL Shortener (Warm-Up)

> **Level:** ★★ intermediate
>
> **You will practise:** the full interview framework from topic 01 on a small classic: clarifying, estimation, API and data model, **key generation** (hash vs counter vs pre-generated), **read-heavy caching**, database choice, **redirect semantics (301 vs 302)**, scaling reads, abuse prevention, and the DevOps layer (deploy, monitor, DR, cost).
>
> **You need:** topics 01–24 (mainly 03, 06, 14–18, 23).
>
> **Part 9 · Case studies** · Previous: [44 · Chaos engineering](44-chaos-engineering-and-game-days.md) · Next: [46 · HA web app](46-case-ha-web-app.md) · [Back to INDEX](../INDEX.md)

---

## How to use a case study

Cover everything below the requirements, set a 45-minute timer, and do it yourself on paper following topic 01's phases. Then compare section by section. Every case study has the same shape:

**Requirements → Estimation → High-level design → Deep dives → Scaling → Reliability & DR → Security → Observability → Cost → Trade-offs → Follow-up questions.**

---

## 1. Requirements and clarifying questions

**Prompt:** "Design a URL shortening service like the ones used in marketing campaigns."

| Question | Assumed answer |
|---|---|
| Core features? | Create short link for a long URL; redirect short → long; optional custom alias; optional expiry |
| Analytics? | Basic click counts per link (async, not real-time precise) |
| Scale? | 100 M new links/month; read:write ≈ 100:1 |
| Latency? | Redirect p99 < 50 ms server-side |
| Availability? | Redirects 99.99% (broken links in printed campaigns are costly); creation 99.9% |
| Link lifetime? | Default 5 years |
| Users? | Public API with API keys + web UI; anonymous creation limited |
| Global? | Users worldwide |

**Functional:** create, redirect, custom alias, expiry, click counts.
**Non-functional:** read-heavy, very low redirect latency, high availability for redirects, durable mappings, abuse protection.

---

## 2. Estimation

```
Writes: 100 M/month ÷ 2.6 M s ≈ 40/s avg  → peak ×3 ≈ 120/s
Reads:  100 × writes ≈ 4,000/s avg        → peak ≈ 12,000/s (campaign spikes can be 10× → plan 40,000/s)

Storage per link: short code 7 B + long URL ~200 B + metadata (owner, created, expiry) ~100 B ≈ 500 B with overhead
5 years: 100 M × 12 × 5 = 6 B links × 500 B = 3 TB  (+ indexes/replicas → ~6–10 TB)

Key space: base62 (a–z, A–Z, 0–9) → 62^7 ≈ 3.5 × 10^12  ≫ 6 × 10^9 needed → 7 characters is plenty

Bandwidth (redirect responses are tiny, ~500 B): 12,000/s × 500 B = 6 MB/s → trivial
Cache: 20% of daily hot links… daily distinct reads maybe 50 M × 500 B × 20% = 5 GB → small Redis cluster
```

**Conclusion:** the hard parts are **unique key generation**, **very fast, highly available reads** (cache + replicated store), and **abuse**. Writes are easy.

---

## 3. High-level design

```
                  ┌──────────── CDN / edge (cache 301s? see deep dive) ─────────────┐
Users ── DNS ──▶  │ WAF + rate limit                                                  │
                  └──────────────┬───────────────────────────────────────────────────┘
                                 ▼
                          [ALB] ─────▶ [Redirect service ×N (stateless)] ──▶ [Redis cache cluster]
                                 │                                    └──miss──▶ [Links DB (key-value)]
                                 └──▶ [API service ×N] ──▶ [Key generator / ID ranges] ──▶ [Links DB]
                                                │
           Redirect service ──click event──▶ [Queue/stream] ──▶ [Analytics workers] ──▶ [Analytics store]
```

**API:**
```
POST /v1/links        {long_url, custom_alias?, expires_at?}   → 201 {short_url}
GET  /{code}          → 302/301 Location: long_url  (or 404/410 if expired)
GET  /v1/links/{code}/stats → {clicks, by_day}
DELETE /v1/links/{code}
```

**Data model (key-value):**
```
links:  code (PK) → long_url, owner_id, created_at, expires_at, status
owners: owner_id → plan, api_key_hash
(index owner_id → codes, for "my links" pages)
```

**Redirect flow:**
1. `GET /aB3xY9k` → CDN/WAF → ALB → redirect service.
2. Redis `GET link:aB3xY9k` → hit (most of the time) → return 302 with `Location`.
3. Miss → DB read → populate cache (TTL 24 h) → 302.
4. Emit click event to a stream **asynchronously** (fire-and-forget with a local buffer); never block the redirect on analytics.

---

## 4. Deep dives

### 4.1 Key generation

| Option | How | Pros | Cons |
|---|---|---|---|
| Hash of long URL (MD5/SHA → base62, take 7 chars) | Deterministic | Same URL → same code (dedup) | Collisions to handle (check-and-retry); dedup not always wanted (per-user analytics) |
| Random 7 chars + check uniqueness | Conditional insert (`PutItem` with "attribute not exists") | Simple, unguessable-ish | Retry on collision (rare: at 6 B of 3.5 T used, ~0.2% chance per try) |
| **Counter + base62** | Global counter → encode | No collisions, short | A single counter is a bottleneck/SPOF; sequential codes are guessable |
| **Counter ranges** ✅ | Each API instance leases a block of 1 M IDs from a coordinator (DB row with atomic increment / ZooKeeper / etcd) | No collisions, no per-request coordination | Unused IDs lost on crash (fine); codes predictable → optionally shuffle with a bijective scramble |

**Choice:** random 7-char codes with conditional insert (simple, unguessable) or ID ranges + scrambling (zero collisions). Custom aliases: conditional insert, 409 if taken; reserved-words list.

### 4.2 301 vs 302

- **301 (permanent):** browsers and CDNs cache it → fewer requests to us, but we **lose click analytics** and can't change/expire the target reliably.
- **302 (temporary):** every click hits us → accurate analytics, changeable targets.
- Choice: **302** with `Cache-Control: private, max-age=0`, or short CDN caching (e.g. 60 s) for very hot links if we accept slightly undercounted analytics (count at the edge via logs instead).

### 4.3 Database choice

Access is **by key**, 6 B records, simple: a **key-value store** (DynamoDB, Cassandra) fits: partitioned by `code` (high cardinality, uniform), single-digit ms reads, multi-AZ replication built in. A sharded Postgres would also work but adds sharding work (topic 18). DynamoDB on-demand/provisioned with auto scaling; global tables if we go multi-region.

### 4.4 Caching

- Redis cluster, cache-aside, TTL 24 h + jitter, ~5–20 GB.
- Hot campaign links: a **local in-process LRU** (5–10 s TTL) in front of Redis to avoid hot-key issues (topic 14).
- **Negative caching** for non-existent codes (30 s) to blunt scanning attacks.

---

## 5. Scaling

- Redirect service: stateless, HPA on requests/s; ~12k–40k req/s at maybe 3–5k req/s per pod → 10–15 pods at peak + N+1 per AZ.
- Redis: 3 shards with replicas; local LRU absorbs hot keys.
- DynamoDB: scales by partitions automatically; key design is uniform.
- **Global latency:** deploy redirect services in 2–3 regions with DynamoDB global tables (writes are rare and owned by the creator; LWW conflicts essentially impossible because codes are unique). Latency-based DNS/anycast routing.
- Analytics: Kinesis/Kafka partitioned by code → workers aggregate per minute → store counts (time-series or DynamoDB counters). Batch writes, not one write per click.

---

## 6. Reliability and DR

- Multi-AZ everything; redirects depend only on cache + DB (analytics is off the critical path).
- If Redis is down: fall back to DB with concurrency limits; DynamoDB handles the read rate.
- If the DB is down: serve from cache (most hot links still work), creation fails gracefully.
- Multi-region active-active for redirects (read-heavy, globally replicated data) → survives region loss; RPO for new links ≈ replication lag (seconds).
- Backups: DynamoDB PITR + periodic export to S3 in another account (topic 41).
- Expired links: TTL attribute deletes them; return 410 Gone.

---

## 7. Security and abuse

- Rate limits: per API key and per IP for anonymous creation (topic 25); CAPTCHA for anonymous web creation.
- **Malicious URLs:** check long URLs against threat feeds (Google Safe Browsing API) at creation and periodically; interstitial warning page or block; report/abuse workflow.
- Unguessable codes for private links (longer codes, e.g. 10+ chars).
- Validate URLs (schemes `http/https` only; no `javascript:`); prevent open-redirect abuse in our own domain.
- WAF, DDoS protection at the edge; API keys hashed; TLS everywhere.

---

## 8. Observability

- **SLIs:** redirect success (non-5xx, valid codes) and latency (< 50 ms server-side) at the ALB; creation success.
- **SLOs:** redirects 99.99% / p99 < 50 ms; creation 99.9%.
- Metrics: cache hit ratio, DB latency/throttles, 404 rate (spikes = scanning), click pipeline lag.
- Synthetic checks: a known short link redirected from 3 regions every minute.

---

## 9. Cost

- Compute: small (tens of pods). Redis: small cluster. DynamoDB: reads mostly served by cache; ~3 TB storage.
- Analytics stream and storage may be the largest item → aggregate early, keep raw click events in S3 (cheap), not in a database.
- Multi-region doubles serving cost; justify by the 99.99% redirect requirement.

---

## 10. Trade-offs summary

| Decision | Chose | Gave up |
|---|---|---|
| 302 over 301 | Accurate analytics, mutable targets | More traffic to us |
| Random codes / ID ranges | No coordination per request | Possible retry (random) or predictability (ranges, mitigated by scrambling) |
| Key-value DB | Simple scale | Ad-hoc queries (analytics elsewhere) |
| Async analytics | Fast, reliable redirects | Counts are eventually consistent |
| Active-active multi-region | Global latency, region resilience | Cost, replication complexity |

---

## 11. Follow-up questions (with answers)

1. **Q: A celebrity posts a link; it gets 200,000 clicks/s. What breaks first?**
   A: The single Redis shard holding that key and the click pipeline partition. The in-process LRU absorbs the key at the redirect pods; CDN short caching (60 s) for that link absorbs most traffic at the edge; click events are sampled/aggregated in-memory per pod before publishing.

2. **Q: How do you prevent two users getting the same code?**
   A: With random codes, a conditional write ("insert only if not exists") makes the DB the arbiter; on conflict, retry with a new code. With ID ranges, uniqueness comes from non-overlapping leased ranges.

3. **Q: How would you deploy changes safely?**
   A: Redirect service is the critical path: canary with analysis on redirect success/latency (topic 34), region by region. Schema is simple; changes additive.

4. **Q: How do you delete links for GDPR requests?**
   A: Delete the mapping, invalidate caches (delete key + short TTL ensures expiry), purge any CDN-cached redirects, and remove or anonymize analytics tied to the owner.
