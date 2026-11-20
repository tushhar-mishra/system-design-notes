# 25 · Rate Limiting and Backpressure

> **Level:** ★★ intermediate
>
> **You will learn:** why and **where** to limit traffic (edge, gateway, service, per tenant, outbound), the algorithms (**token bucket, leaky bucket, fixed window, sliding window log, sliding window counter**) with their trade-offs, **distributed rate limiting** with Redis, **local vs global** limits, what to return (**429 vs 503**, `Retry-After`, rate-limit headers), **fail open vs fail closed**, **quotas** vs **rate limits**, and **backpressure** through the whole system.
>
> **You need:** topics 06, 10, 14, 21, 24.
>
> **Part 4 · Reliability foundations** · Previous: [24 · Resilience patterns](24-resilience-patterns.md) · Next: [26 · Distributed systems realities](26-distributed-systems-realities.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **rate limiter** caps how many requests a client (user, API key, IP, tenant) may make in a time window. It protects your system from overload, abuse, noisy neighbours and runaway scripts, and enforces fair use and paid tiers.

**Token bucket, the default algorithm:**
```
Bucket holds up to B tokens (burst). Tokens refill at R per second.
Each request takes 1 token. No token → reject with 429.
e.g. R = 10/s, B = 20: a client can burst 20 at once, then sustain 10/s.
```

**Picture it.** An arcade token machine: it drops one token into your cup every second, and your cup holds at most 20. Each game costs a token. You can play 20 games quickly if you saved up, but then only one per second.

**The one thing to know.** Return **429 Too Many Requests** with **`Retry-After`** so well-behaved clients back off. Use **503** for "*we* are overloaded" (load shedding, topic 24) and **429** for "*you* are over your limit".

**Try it now:** `curl -sI https://api.github.com/users/octocat | grep -i ratelimit`: GitHub's limit, remaining, and reset time.

**You've got it if you can:** explain token bucket vs sliding window, design a distributed limiter with Redis, and choose fail-open vs fail-closed.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **Rate limit:** max requests per time window (e.g. 100/min). Short-term protection.
- **Quota:** max usage per long period (e.g. 1 M API calls/month). Business/billing.
- **Concurrency limit:** max requests **in flight** at once (protects against slow, expensive requests better than a rate limit).
- **Throttling:** slowing or rejecting requests above a limit (cloud APIs "throttle" you).
- **Backpressure:** a slow consumer signalling upstream to slow down (bounded queues, 429/503, TCP flow control), instead of being buried.

**Where limits live:**

```
Internet → [CDN/WAF: per-IP, bot rules] → [API gateway: per API key/tenant/plan]
        → [Service: per-user/endpoint, concurrency limits] → [DB pool: max connections]
Service → [Outbound limiter: respect third-party limits, e.g. 100 req/s to payment API]
```

---

## Why it exists

1. **Protect availability:** one client's bug (an infinite loop calling your API) shouldn't take you down for everyone.
2. **Fairness / noisy neighbour:** in multi-tenant systems, one big tenant can't starve others.
3. **Security:** slow brute-force login attempts, scraping, credential stuffing, DDoS at L7.
4. **Cost control:** expensive endpoints (search, AI calls, exports) and pay-per-use downstreams.
5. **Business:** pricing tiers (free 10 req/s, premium 200 req/s).
6. **Respect others' limits:** cloud APIs (AWS, GitHub) throttle you; outbound limiting avoids being blocked.

---

## Analogy

Arcade token machine (Core). Others:
- **Leaky bucket** = a funnel with a small hole: you can pour fast, but water exits at a steady rate; if the funnel overflows, the extra spills (rejected).
- **Fixed window** = "100 entries per hour, counter resets on the hour": a crowd can enter 100 at 9:59 and 100 more at 10:00.

---

## How it works inside

### Algorithms

| Algorithm | How | Pros | Cons |
|---|---|---|---|
| **Token bucket** | Tokens refill at rate R up to capacity B; request consumes a token | Allows controlled bursts; O(1) memory per key; most common (AWS API Gateway, Envoy, many libs) | Two parameters to tune |
| **Leaky bucket** (as queue) | Requests enter a queue drained at fixed rate | Smooth output rate | Adds latency; bursts queue up |
| **Fixed window counter** | Count per key per window (`key:10:00`) | Simplest, cheap | Burst of 2× at window edges |
| **Sliding window log** | Store timestamps of requests; count those in last N s | Exact | Memory per request (expensive at scale) |
| **Sliding window counter** | Weighted sum of current and previous fixed windows | Close to exact, cheap | Approximation |
| **Concurrency limiter** | Count in-flight; reject above N | Protects from slow/expensive requests | Doesn't cap rate of cheap requests |

**Sliding window counter example:** limit 100/min. At 10:00:15, previous window (09:59) had 80, current (10:00) has 30. Weight of previous = (60 − 15)/60 = 0.75. Estimate = 80 × 0.75 + 30 = 90 → allowed (< 100).

### Distributed rate limiting (many gateway replicas)

Each replica counting locally would let a client get N × limit (one limit per replica). Options:

```
Option A — central store (Redis):
   gateway replica ──(atomic script: refill + take token for key "apikey:abc")──▶ Redis
   ← allowed / denied (+ remaining, reset)
   ~0.5–1 ms extra per request; Redis is now on the hot path (HA + timeouts needed)

Option B — local limits with a share:
   limit 1,000/s across 10 replicas → each enforces 100/s locally (no network)
   inaccurate if traffic is uneven across replicas; simple and fast

Option C — hybrid:
   local token buckets that sync counts to Redis every ~100 ms (approximate, cheap)
```

**Atomicity:** the read-modify-write must be atomic, using a **Lua script** in Redis (or `INCR` + `EXPIRE` for fixed windows), otherwise concurrent requests race past the limit.

```lua
-- token bucket in Redis (sketch): KEYS[1]=bucket key, ARGV = rate, capacity, now, cost
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1]) or tonumber(ARGV[2])
local ts     = tonumber(b[2]) or tonumber(ARGV[3])
tokens = math.min(tonumber(ARGV[2]), tokens + (tonumber(ARGV[3]) - ts) * tonumber(ARGV[1]))
local allowed = tokens >= tonumber(ARGV[4])
if allowed then tokens = tokens - tonumber(ARGV[4]) end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
redis.call('EXPIRE', KEYS[1], 3600)
return allowed and 1 or 0
```

Shard keys across a Redis Cluster (by API key) so no single node is hot. For a global multi-region limit, accept approximate limits (per-region budgets) rather than cross-region calls on every request (topic 55).

### Responses

```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 12
```
(`RateLimit-*` headers follow an IETF draft; `X-RateLimit-*` is the older common form.)

Clients should back off using `Retry-After` + jitter (topic 24).

### Choosing keys and limits

- **Key:** API key / user ID / tenant (authenticated) > IP (shared NAT and IPv6 make IP-based limits crude; use for unauthenticated endpoints and abuse).
- **Per endpoint cost:** a search costs 50× a GET by ID → separate limits, or **weighted** tokens (cost per request).
- **Set limits from capacity:** total sustainable capacity ÷ expected active clients, with burst allowances; leave headroom.
- **Shadow mode first:** log "would have limited" for a week before enforcing, to avoid blocking real customers.

### Fail open vs fail closed

If the limiter's store (Redis) is down:
- **Fail open** (allow): availability first; risk of overload/abuse during the outage. Common for general API limits, ideally with a **local fallback limiter** (approximate per-replica limit).
- **Fail closed** (deny): safety first. For security-critical limits (login attempts, OTP verification, expensive actions).

### Backpressure end to end

```
Users → Gateway (429 per client) → Service (503 + concurrency limit) → Queue (bounded; producer blocked/slowed)
      → Workers (scale up to max) → DB (connection pool limit)
Every layer has a limit; overload is pushed back to the edge where it's cheapest to reject.
```

Unbounded anything (queues, thread pools, connection pools, retries) breaks backpressure: work piles up silently until memory or latency explodes.

---

## Options and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| Algorithm | Token bucket / sliding window / fixed window | Token bucket for APIs with bursts; fixed window for simple quotas; sliding window when edge bursts matter |
| Location | Edge/WAF / gateway / service / outbound | Defence in depth: coarse at edge (IP), precise at gateway (key/tenant), concurrency in service |
| Accuracy | Global exact (central store) / local approximate | Exact for billing tiers and security; approximate for protection |
| Store failure | Fail open / closed | Open + local fallback for general; closed for security |
| Managed vs self | AWS WAF rate rules, API Gateway usage plans, Cloudflare / Envoy + Redis, Kong | Managed for simple per-IP/key; self-built for complex tenant logic |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Limits too low** | Legit customers get 429s | 429 rate by customer, support tickets | Shadow mode, per-tier review, dashboards per key |
| **Limits per replica** | Effective limit = N × configured; changes when scaling | Clients exceed expected rate | Central or shared-budget limiter |
| **Limiter store on hot path fails** | All requests slow/fail (if no timeout) | Redis latency/errors | Short timeouts, fail-open with local fallback, Redis HA |
| **Hot key in limiter store** | One Redis shard saturated (one huge tenant) | Per-shard CPU | Local pre-limiting, split keys |
| **IP-based limit blocks a whole company** (shared NAT) | One office/university blocked | 429s by IP with many users | Authenticated keys; higher IP limits |
| **Clients retry immediately on 429** | Load doesn't drop | Repeated 429 from same key | `Retry-After`, client SDK backoff, escalate penalty |
| **No outbound limiting** | Third party blocks you (their 429s/ban) | 429s from provider | Outbound token bucket per provider, queue + workers |
| **Cloud API throttling during incident** | Terraform/autoscaling/controllers fail with `Throttling` | API error metrics | Backoff, reduce polling, request quota increases |

**3 AM:** "A customer's integration bug is sending 50× normal traffic, latency for everyone is up." Apply a tight limit to that key at the gateway (or block it temporarily), contact the customer. Afterwards: per-key limits by default for every plan.

---

## Where you'll use it

- **Gateways and Ingress:** nginx `limit_req` (leaky bucket), Envoy local/global rate limit, Kong plugins (topic 10).
- **Kubernetes API server:** API Priority and Fairness is a built-in concurrency/fairness limiter; controllers must respect it.
- **CI/CD:** GitHub API rate limits for bots and Actions ([GHA 41 · billing and limits](../../GitHub-Actions-Basics-To-Production/notes/41-billing-limits-and-cost.md)); Docker Hub pull rate limits → pull-through cache.
- **Terraform:** `-parallelism` lowers concurrent API calls to avoid cloud throttling.
- **Case study 55:** full distributed rate limiter design.

---

## Design exercise

**Prompt:** A public API with 3 plans: free (5 req/s, burst 10), pro (50 req/s, burst 100), enterprise (custom). 40 gateway replicas across 2 regions. Login endpoint must resist credential stuffing. Design the limiting.

<details>
<summary><b>Model answer</b></summary>

- **Edge (CDN/WAF):** per-IP rate rules (e.g. 2,000 req/5 min per IP for all endpoints), bot management, geo blocks if needed. Coarse protection.
- **Gateway, per API key, per region:** token bucket in a **regional Redis Cluster** (Multi-AZ), Lua script, key `rl:{apikey}`. Plan → rate/burst from a cached plan table. Each region enforces the full limit for keys whose home region it is, and a smaller share for traffic arriving elsewhere (approximate global limit; avoids cross-region round trips). Headers + 429 + `Retry-After`.
- **Weighted costs:** `/search` costs 5 tokens, `/export` uses a separate concurrency limit (2 per key).
- **Failure:** Redis timeout 5 ms → fail open to a **local per-replica bucket** at (limit ÷ replicas in region × 2); alert.
- **Login:** separate limiter, **fail closed**: per account (5 failed attempts / 15 min → backoff/CAPTCHA), per IP, and global anomaly detection on failed-login rate. Limits keyed by username too, since stuffing rotates IPs.
- **Rollout:** shadow mode for a week, then enforce; dashboards of 429s by plan and key; customer-facing docs on limits.
- **Outbound:** if our API calls a third-party per request, a separate outbound limiter + queue protects that provider's limits.
</details>

---

## Quick recap

- Rate limits protect availability, fairness, security and cost; quotas are long-term; concurrency limits protect from slow requests.
- Token bucket (rate + burst) is the default; sliding window counter is a cheap accurate alternative; fixed window bursts at edges.
- Distributed limits need a shared atomic store (Redis + Lua) or approximate local shares.
- 429 + `Retry-After` for client over limit; 503 for server overload. Fail open (general) vs closed (security).
- Backpressure: bounded everything, push overload to the edge.

### Interview questions

1. **Q: Token bucket vs fixed window: why prefer token bucket?**
   A: Fixed windows reset abruptly, so a client can send the full limit at the end of one window and again at the start of the next (2× burst). Token bucket enforces a steady average rate with an explicit, bounded burst, using constant memory per key.

2. **Q: How do you enforce a per-customer limit across 50 gateway instances?**
   A: Use a shared store: an atomic Lua token-bucket script in a sharded Redis keyed by customer, with tight timeouts and a local fallback limiter if Redis is unreachable. If a little inaccuracy is acceptable, split the budget across instances or sync local counters periodically to avoid a network hop on every request.

3. **Q: Your rate limiter's Redis is down. Allow or deny traffic?**
   A: Depends on the limit's purpose. For general API protection, fail open with an approximate local limit so a limiter outage doesn't become an API outage. For security limits like login attempts or OTP checks, fail closed (or require extra verification), because unlimited attempts are the bigger risk.
