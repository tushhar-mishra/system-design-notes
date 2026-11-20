# 14 · Caching Fundamentals and Patterns

> **Level:** ★★ intermediate
>
> **You will learn:** what a **cache** is and **where caches live** (browser, CDN, proxy, app memory, distributed cache, database buffer), the **hit ratio** math, the main patterns (**cache-aside, read-through, write-through, write-behind, write-around**), **TTL**, **eviction** (LRU, LFU), **invalidation** (the hard part), and the failures that page you: **stampede / thundering herd, hot keys, cold cache after restart, stale data, cache becoming a hard dependency**. Plus **Redis vs Memcached** and how to run them reliably.
>
> **You need:** topics 02–03, 11.
>
> **Part 3 · Caching and data** · Previous: [13 · Monolith vs microservices](13-monolith-vs-microservices.md) · Next: [15 · CDN and edge](15-cdn-and-edge.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **cache** keeps a copy of data in a faster place (usually memory) so you don't redo slow work (a DB query, an API call, a computation). Memory is ~1,000× faster than a network+disk round trip (topic 03).

**Cache-aside, the default pattern:**
```
read(key):
  value = cache.get(key)
  if value is missing:                ← "miss"
      value = db.query(key)
      cache.set(key, value, ttl=300s)
  return value

write(key, value):
  db.update(key, value)
  cache.delete(key)                   ← invalidate; next read refills
```

**Picture it.** A chef keeps the 20 most-used spices on the counter (cache) instead of walking to the storeroom (database) every time. When a spice jar is replaced with a new batch in the storeroom, the old counter jar must be thrown away (invalidation), or the chef keeps using the old one (stale data).

**The one thing to know.** The app must **survive the cache disappearing**. If the cache fails or restarts empty and all traffic hits the database at once, the DB falls over (stampede). Design for the cold cache.

**Try it now:** `redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"` → hit ratio = hits / (hits + misses).

**You've got it if you can:** explain cache-aside and write-through, choose a TTL, and describe three ways to prevent a cache stampede.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**Cache** = a temporary, fast copy of data whose **source of truth** lives elsewhere. Losing the cache must never lose data (unless you use write-behind, below).

Terms:
- **Hit** = data found in cache. **Miss** = not found, go to the source.
- **Hit ratio** = hits ÷ total lookups.
- **TTL** (Time To Live) = how long an entry stays before it expires.
- **Eviction** = removing entries when the cache is full.
- **Invalidation** = removing/updating entries because the source changed.
- **Stale data** = cache holds an old version.
- **Warm cache** = full of useful data. **Cold cache** = empty (after restart/deploy).

**Where caches live (from user to disk):**

| Layer | Example | Caches |
|---|---|---|
| Browser | `Cache-Control` headers | Static files, API GETs |
| CDN / edge | CloudFront, Cloudflare | Static and some dynamic responses (topic 15) |
| Reverse proxy | nginx `proxy_cache`, Varnish | Whole HTTP responses |
| App in-process | A map/LRU in app memory (Caffeine, lru_cache) | Hot, small, rarely changing data |
| **Distributed cache** | **Redis, Memcached** (ElastiCache, Memorystore, Azure Cache for Redis) | Query results, sessions, computed objects |
| Database | Buffer pool / page cache | Data pages in RAM |
| OS | Linux page cache ([Linux 093 · memory](../../linux-mastery-rhel/notes/093-memory-management.md)) | File blocks |

---

## Why it exists

1. **Latency:** ~0.5 ms from Redis vs 5–50 ms for a DB query or 100+ ms for a remote API.
2. **Load reduction:** a 95% hit ratio means the DB sees only 5% of reads: **20× fewer** queries.
3. **Cost:** cache memory is cheaper than scaling a database to the same read throughput.
4. **Resilience** (sometimes): serve stale data while the source is down.

**Hit ratio math:**
```
Average latency = hit_ratio × cache_latency + (1 − hit_ratio) × db_latency
90% hits, 1 ms cache, 20 ms DB → 0.9×1 + 0.1×20 = 2.9 ms
99% hits                      → 0.99×1 + 0.01×20 = 1.2 ms
DB load: 10,000 reads/s × (1 − 0.9) = 1,000/s;  at 99% → 100/s
```
Going from 90% to 99% cuts DB load **10×**. Small hit-ratio changes matter a lot, which is why a cache restart is so dangerous.

---

## Analogy

The chef's counter (Core). Extras:
- **TTL** = "throw away counter jars after a week, whatever happens".
- **LRU eviction** = counter is full, so put away the spice not used for the longest time.
- **Stampede** = the counter gets wiped clean during dinner rush and 20 cooks run to the storeroom at the same moment, blocking the door.
- **Hot key** = every dish needs salt; one salt jar, 20 cooks queueing for it.

---

## How it works inside

### The patterns

```
Cache-aside (lazy loading)           Read-through
App ─get─▶ Cache (miss)              App ─get─▶ Cache ─(miss: cache itself loads)─▶ DB
App ─query─▶ DB                      (cache library/provider knows how to load)
App ─set─▶ Cache

Write-through                        Write-behind (write-back)
App ─write─▶ Cache ─sync─▶ DB        App ─write─▶ Cache ─(later, batched, async)─▶ DB
(both updated before ack)            ⚠️ cache crash = lost writes

Write-around
App ─write─▶ DB only; cache filled on next read (cache-aside)
```

| Pattern | Reads | Writes | Pros | Cons | Use when |
|---|---|---|---|---|---|
| **Cache-aside** | App checks cache, loads on miss | App writes DB, deletes cache key | Simple, cache failure ≠ data loss, only cached what's read | First read is a miss; race conditions possible | **Default** |
| **Read-through** | Cache loads on miss | — | Cleaner app code | Needs cache-side loader | Libraries/providers that support it |
| **Write-through** | Always fresh | Write cache + DB synchronously | Cache always up to date | Slower writes; caches data never read | Read-after-write consistency matters |
| **Write-behind** | Fast | Cache acks, DB updated later | Very fast writes, batching | **Data loss risk**, complex | Counters, analytics where loss is OK |
| **Write-around** | — | DB only | Doesn't pollute cache with write-once data | Read after write is a miss | Write-heavy, rarely re-read data (logs) |

### Invalidation: delete, don't update

On write, **delete** the cache key rather than **setting** the new value. Why: two concurrent writers can set values in the wrong order, leaving the older value in cache forever. Deleting means the next reader reloads the truth.

Even delete-on-write has a race (reader loads old value from DB, writer updates DB and deletes key, reader then sets the old value). **TTL is the safety net**: it bounds how long any wrong value can live. Always set a TTL.

Other invalidation tools:
- **Versioned keys:** `product:42:v17`; bump the version on change; old keys expire naturally.
- **Event-driven:** DB change events (CDC, topic 13) publish invalidations.
- **Short TTL only:** accept staleness of N seconds; simplest for many cases.

### TTL choice

| Data | TTL idea |
|---|---|
| Product catalog, config | Minutes to hours |
| User profile | Minutes, delete on update |
| Prices, stock levels | Seconds, or don't cache the decision point (checkout reads DB) |
| Rendered public pages | Seconds to minutes (CDN) |
| Session | Session lifetime |

Add **jitter** (random ±10%) to TTLs so many keys created together don't all expire at the same second.

### Eviction policies

- **LRU** (Least Recently Used): evict what hasn't been read for longest. Good default.
- **LFU** (Least Frequently Used): evict what's read least often. Better for stable popularity.
- **TTL-based / random.**
- Redis: `maxmemory-policy` (`allkeys-lru`, `allkeys-lfu`, `volatile-lru` (only keys with TTL), `noeviction` → writes fail when full ⚠️).

### Stampede (thundering herd) protection

When a hot key expires (or the cache restarts), thousands of requests miss together and all hit the DB.

| Technique | How |
|---|---|
| **Request coalescing / single-flight** | Only one request per key loads from DB; others wait for its result (in-process lock, or a short Redis lock `SET key:lock NX EX 5`) |
| **Stale-while-revalidate** | Serve the expired value while one background refresh runs |
| **Probabilistic early refresh** | Each read has a small chance to refresh before expiry, rising as TTL nears 0 |
| **TTL jitter** | Spread expiries |
| **Pre-warming** | Load hot keys before switching traffic to a new cache |
| **Protect the DB** | Connection pool limits + load shedding so the DB degrades instead of collapsing |

### Hot keys

One key read 100,000 times/s → one Redis shard (single-threaded command execution) saturates.
- Local in-process cache for that key (1–5 s TTL) in front of Redis.
- Replicate the key: `hotkey#1..#N`, read a random copy.
- Read from replicas.

### Redis vs Memcached

| | Redis / Valkey | Memcached |
|---|---|---|
| Data types | Strings, hashes, lists, sets, sorted sets, streams | Strings only |
| Persistence | Optional (RDB snapshots, AOF log) | None |
| Replication & failover | Yes (replicas, Sentinel, Cluster mode) | No (client-side sharding) |
| Threads | Single-threaded command execution (I/O threads help) | Multi-threaded |
| Extra uses | Rate limiting, locks, queues, leaderboards, pub/sub | Pure cache |
| Choose | Default today | Simple, very large, multi-threaded pure caching |

> **Valkey** = the open-source fork of Redis created in 2024 after Redis changed its license; AWS ElastiCache and others support it. Same protocol.

Running it: **cluster mode** shards keys across nodes (16,384 hash slots); each shard has a primary + replica in another AZ with automatic failover. Monitor memory, evictions, hit ratio, CPU, replication lag, connected clients.

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| In-process vs distributed | In-process: fastest (ns), no network, but each instance has its own copy (inconsistency across instances; memory × N) | Distributed: shared, consistent across instances, ~0.5 ms network hop, another system to run |
| Short vs long TTL | Short: fresher, more DB load | Long: less load, more staleness |
| Cache everything vs targeted | Everything: memory waste, invalidation complexity | Targeted: cache only hot, expensive, read-heavy data ✅ |
| Fail open vs closed when cache is down | Open: go to DB (risk: DB overload) | Closed: return error/degraded (safe for DB, bad UX) → usually open **with** DB protection (limits, shedding) |

**Don't cache when:** data is written more than read, every request is unique, strong consistency is required at that point (bank balance at transfer time), or the source is already fast enough.

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Cache cluster down / restarted empty** | DB CPU spikes to 100%, latency up, maybe total outage | Hit ratio ↓, DB load ↑ together | Replicas + failover, stampede protection, DB limits, gradual warm-up |
| **Stampede on hot key expiry** | Periodic DB spikes | DB spikes aligned with TTL | Single-flight, stale-while-revalidate, jitter |
| **Hot key** | One shard at 100% CPU, others idle | Per-shard CPU skew, `redis-cli --hotkeys` | Local cache, key replication |
| **Stale data** | Users see old prices/profile | Complaints, consistency checks | Delete-on-write, shorter TTL, versioned keys |
| **Cache memory full with `noeviction`** | Writes fail: `OOM command not allowed` | Memory %, rejected writes | `allkeys-lru`, size properly, TTLs on everything |
| **Big keys / slow commands** (`KEYS *`, huge lists) | Latency spikes for all clients (single thread blocked) | `SLOWLOG`, latency monitor | Ban `KEYS` (use `SCAN`), limit value sizes |
| **Cache penetration** (requests for keys that don't exist, e.g. attack with random IDs) | All miss → DB | Miss ratio high for unknown keys | Cache negative results briefly, bloom filter, rate limit |
| **Cache as hidden hard dependency** | Cache outage = full outage | Game day reveals it | Timeouts on cache calls (e.g. 50 ms), fallback to DB with limits |
| **Thundering herd after deploy** (in-process caches all cold) | Spike after each rollout | DB load at deploy time | Gradual rollout, pre-warm |

**3 AM:** "DB CPU at 100%, everything slow, nothing deployed." Check cache hit ratio and Redis health first: a failover or node replacement emptied part of the cache. Mitigate: rate-limit / shed load at the edge, raise DB capacity temporarily, let the cache refill gradually. Prevent: stampede protection and DB connection limits.

---

## Where you'll use it

- **Web apps:** product, profile and config caching in Redis (case study 46).
- **CI/CD:** dependency caches and Docker layer caches are the same idea ([GHA 25 · caching](../../GitHub-Actions-Basics-To-Production/notes/25-caching.md), [Docker 16 · build cache](../../Docker/notes/16-buildkit-and-build-cache.md)).
- **Ansible fact caching** ([Ansible 48](../../Ansible/notes/48-performance-tuning.md)) and **Terraform provider plugin cache**: caching to avoid repeated slow work.
- **Kubernetes:** API server watch cache; NodeLocal DNSCache (topic 07).
- **Rate limiting and locks** on Redis (topics 25, 26).

---

## Design exercise

**Prompt:** A product page reads product details (changes a few times a day), price (changes every few minutes), and stock (changes on every order). 20,000 reads/s at peak; Postgres handles ~3,000 queries/s comfortably. Design caching.

<details>
<summary><b>Model answer</b></summary>

- **Product details:** Redis cache-aside, TTL 1 h + jitter, **delete on update** (the admin service deletes the key after DB commit). Also CDN-cacheable if the page is public (topic 15).
- **Price:** Redis, TTL 60 s, delete on price change. At checkout, the **order service reads price from the DB** (source of truth at the decision point).
- **Stock:** don't cache exact numbers for checkout. For display, cache a coarse "in stock / low / out" with a 10–30 s TTL. Reservation at checkout uses the DB with a conditional update (`UPDATE ... SET stock = stock - 1 WHERE id = ? AND stock > 0`).
- **Load math:** 20,000 reads/s × 3 lookups = 60,000 cache lookups/s → a Redis cluster with 3 shards (each well under ~100k ops/s), each with a replica in another AZ. With ~98% hit ratio, DB sees ~1,200 queries/s → within its 3,000 budget with headroom.
- **Stampede:** single-flight per key in the app; stale-while-revalidate for product details; TTL jitter.
- **Cache down:** timeouts of ~50 ms on Redis calls; fall back to DB **through a concurrency limiter** (e.g. max 200 in-flight DB queries per service) so the DB degrades, not collapses; alert on hit ratio < 90%.
</details>

---

## Quick recap

- Cache = fast copy; source of truth elsewhere. Hit ratio drives everything (90% → 99% = 10× less DB load).
- Cache-aside + delete-on-write + TTL (with jitter) is the default.
- Write-through for freshness; write-behind only when losing writes is acceptable.
- Plan for stampedes, hot keys, cold starts, and cache outages: the DB must survive a cold cache.
- Redis/Valkey is the default distributed cache; run it with replicas across AZs and monitor hit ratio, memory, evictions.

### Interview questions

1. **Q: Why delete the cache key on write instead of updating it?**
   A: Updating can race: two writers may set values in a different order than they committed to the DB, leaving an old value cached indefinitely. Deleting forces the next reader to load the committed value. A remaining read-write race is bounded by the TTL, which is why every key should have one.

2. **Q: Your Redis primary failed over and the site went down even though Redis came back in 30 seconds. Why?**
   A: Probably the cache became a hard dependency: during failover, calls hung without short timeouts, filling app threads; or the new primary had lost recent data and the miss storm overloaded the DB. Fix with tight timeouts on cache calls, fallback paths with concurrency limits to protect the DB, stampede protection, and load shedding.

3. **Q: When would you not add a cache?**
   A: When reads are rarely repeated, writes dominate, the data must be strongly consistent at that point (balances at transfer time), or the source is already fast enough. A cache adds staleness, invalidation bugs and another failure mode, so I add it for hot, expensive, read-heavy data with a measured need.
