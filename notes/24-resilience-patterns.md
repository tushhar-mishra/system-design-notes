# 24 · Resilience Patterns

> **Level:** ★★★ advanced
>
> **You will learn:** how to stop one slow or failing component from taking everything down: **timeouts** (and how to set them), **retries** with **exponential backoff + jitter** and **retry budgets**, **circuit breakers**, **bulkheads**, **load shedding** and **admission control**, **graceful degradation** and **fallbacks**, **health-check and dependency hygiene**, **cascading failures** and **metastable failures** (retry storms that don't recover on their own), and where each pattern lives (code, mesh, LB, gateway).
>
> **You need:** topics 04, 06, 09, 10, 22, 23.
>
> **Part 4 · Reliability foundations** · Previous: [23 · Availability](23-availability-and-the-nines.md) · Next: [25 · Rate limiting](25-rate-limiting-and-backpressure.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Failures spread. A slow database makes app threads wait, waiting threads fill the pool, callers time out and retry, retries add load, and the whole system collapses: a **cascading failure**. Resilience patterns **contain** failures.

```
Timeout          → don't wait forever
Retry + backoff  → try again, but politely (exponential delay + random jitter), limited
Circuit breaker  → stop calling a dependency that's failing; fail fast; probe later
Bulkhead         → separate pools so one dependency can't use all threads/connections
Load shedding    → reject excess requests early (fast 503/429) to protect the core
Degradation      → serve a reduced experience instead of an error
```

**Picture it.** A ship. **Bulkheads** are watertight walls: a hole floods one compartment, not the ship. A **circuit breaker** is the fuse in your house: a short circuit trips it instead of burning the house. **Load shedding** is a nightclub door policy: when full, new people wait outside instead of crushing everyone inside.

**The one thing to know.** **Retries multiply load exactly when the system is weakest.** 3 retries at each of 3 layers = up to 64× (4³) the original traffic hitting the bottom layer. Retry at **one** layer, with backoff, jitter and a budget.

**Try it now:** find the HTTP client config in a service you know. Is there a timeout? (Many libraries default to **no timeout** or very long ones.)

**You've got it if you can:** set timeouts for a call chain, explain backoff with jitter, and describe how a circuit breaker changes states.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Pattern | Protects | Lives in |
|---|---|---|
| **Timeout** | Caller's resources (threads, connections) | Every client call, LBs, gateways, DB drivers |
| **Retry with backoff + jitter** | Against transient failures | One layer per call path (client lib or mesh) |
| **Retry budget** | Dependency from retry storms | Client libs, Envoy/mesh (`retry_budget`) |
| **Circuit breaker** | Caller (fail fast) and dependency (rest) | Client libs (resilience4j, Polly), mesh (outlier detection) |
| **Bulkhead** | Isolation between dependencies/tenants | Separate thread/connection pools, separate deployments, per-tenant quotas |
| **Load shedding / admission control** | The service itself under overload | Server middleware, gateway, LB, queue limits |
| **Graceful degradation / fallback** | User experience | App logic |
| **Deadline propagation** | Avoid work nobody waits for | gRPC deadlines, request headers |
| **Idempotency** | Safe retries (topics 06, 22) | API design |

---

## Why it exists

In a distributed system something is always partly broken (topic 23). Without containment, **partial failure becomes total failure**: Amazon, Google and others publish postmortems where a small slowdown cascaded through retries and resource exhaustion into hours-long outages. These patterns are cheap insurance and a staple of senior interviews.

---

## Analogy

Ship bulkheads, house fuses, nightclub door (Core). Extras:
- **Timeout** = hanging up after 30 seconds on hold instead of waiting all day.
- **Backoff + jitter** = after a busy signal, wait 1 minute, then 2, then 4, each time plus a random few seconds, so everyone doesn't redial at the same instant.
- **Degradation** = the restaurant is out of the special, so they serve the regular menu rather than closing.

---

## How it works inside

### 1. Timeouts

```
User ──(edge timeout 30 s)──▶ Gateway ──(10 s)──▶ Service A ──(2 s)──▶ Service B ──(500 ms)──▶ DB
```

Rules:
- **Every network call has a timeout.** Connect timeout (short: 0.5–2 s) + request timeout.
- **Set from data:** e.g. dependency p99.9 latency × 1.5–2, not a round number pulled from the air.
- **Outer > inner:** each layer's timeout must exceed the time inner layers may take (including one retry), otherwise outer layers give up while inner ones keep working (wasted work, 504s).
- **Deadline propagation:** pass the remaining time budget down (gRPC deadlines do this) so deep services stop work the caller has abandoned.

### 2. Retries, backoff, jitter, budgets

```
delay = min(cap, base × 2^attempt)          → 100 ms, 200, 400, 800 ... capped at e.g. 5 s
full jitter: sleep = random(0, delay)       → spreads clients so they don't retry in sync
```

- Only retry **retryable** errors (timeouts, connection errors, 503, 429 with `Retry-After`), and only **idempotent** operations (or with idempotency keys).
- **Limit attempts** (2–3 total) and **retry at one layer** (usually the closest to the failing dependency, or the mesh), not at every hop.
- **Retry budget:** allow retries to add at most ~10–20% extra load; when exceeded, stop retrying. This turns a retry storm into a few failures.
- Without jitter, all clients retry at the same moments → synchronized waves ("thundering herd").

**Retry amplification math:**
```
3 layers, each does 1 try + 3 retries (4 attempts):
bottom layer sees up to 4 × 4 × 4 = 64 requests per user request
```

### 3. Circuit breaker

```
          failures ≥ threshold (e.g. 50% of last 20 calls, or 5 consecutive)
 CLOSED ───────────────────────────────────────────▶ OPEN
 (calls flow)                                         (fail fast, don't call; return fallback)
   ▲                                                    │ after cool-down (e.g. 30 s)
   │ trial calls succeed                                ▼
   └──────────────────────────────────────────── HALF-OPEN (let a few trial calls through)
                       trial calls fail → back to OPEN
```

Benefits: callers stop wasting threads on a dead dependency (fast failure), and the dependency gets breathing room to recover. Mesh equivalent: **outlier detection** ejects individual bad hosts from the pool (Envoy/Istio `outlierDetection`).

### 4. Bulkheads

```
Without bulkhead: one pool of 200 threads shared by calls to Payments, Search, Recommendations
   Recommendations hangs → all 200 threads stuck waiting on it → payments fail too

With bulkheads:   Payments pool 100 | Search pool 60 | Recommendations pool 40
   Recommendations hangs → only its 40 threads stuck; payments fine
```

Bulkheads at bigger scale: separate deployments for critical vs non-critical paths, separate node pools, per-tenant quotas, separate DB connection pools for web vs batch, **cells** (topic 42).

### 5. Load shedding and admission control

When overloaded, a server that accepts everything gets slower for **everyone** (queues grow, latency explodes, timeouts, retries → collapse). Better: reject some work **early and cheaply**.

- Limit concurrency (max in-flight requests); return `503` + `Retry-After` beyond it.
- **Prioritize:** shed low-priority traffic first (bots, background sync, analytics) before checkout/login.
- **Drop work whose deadline already passed** (the client gave up).
- **Adaptive concurrency limits** (e.g. Netflix concurrency-limits, Envoy adaptive concurrency) adjust the limit based on observed latency.
- Bounded queues everywhere (unbounded queues = hidden latency, topic 21).

### 6. Graceful degradation and fallbacks

| Dependency down | Degraded behaviour |
|---|---|
| Recommendations | Show "popular items" from cache/static list |
| Personalization | Generic page |
| Search | Browse categories; cached top searches |
| Reviews | Hide the section |
| Fraud scoring | Rule-based fallback with lower limits (topic 23) |
| Feature-flag service | Use last known flags / safe defaults (topic 57) |

Decide **in advance** which features are critical and which can be dropped (a "degradation plan"), and test it.

### Metastable failures

A system can enter a bad state that **persists after the trigger is gone**: e.g. a 10-second DB blip causes timeouts → clients retry → load stays above capacity → requests keep timing out → retries continue, forever. Recovery requires **removing load** (shed traffic, disable retries, scale significantly, or block at the edge) not just fixing the original issue. Retry budgets, backoff, load shedding and circuit breakers prevent entering it.

### Cascading failure: the flow

```
1. DB slows (lock, bad query)               p99 20 ms → 2 s
2. App threads wait on DB                    pool exhausts
3. App can't serve even DB-free endpoints    health checks time out → LB removes instances
4. Remaining instances get more traffic      overload → more removals
5. Clients/gateway retry                     load × 3
6. Autoscaler adds pods                      more DB connections → DB worse
7. Total outage
Breakers: DB timeout 500 ms (2), bulkhead pools (3), shallow health checks (3), load shedding (4),
          retry budget (5), connection limits/pooler (6)
```

---

## Options and trade-offs

| Pattern | Benefit | Cost / risk |
|---|---|---|
| Aggressive timeouts | Free resources fast | False failures if set below normal p99 |
| Retries | Hide transient errors | Amplify load; duplicates if non-idempotent |
| Circuit breaker | Fail fast, protect dependency | Tuning; may trip on a partial issue and fail healthy calls |
| Bulkheads | Isolation | Lower utilization (reserved capacity) |
| Load shedding | Survive overload | Some users get errors (deliberately) |
| Degradation | Users keep core function | Engineering effort, testing, product decisions |
| In code (libraries) | Fine-grained, app-aware fallbacks | Per-language implementations |
| In mesh/gateway | Uniform, no code changes | Can't do app-specific fallbacks; more config to manage |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Missing timeout** | Threads hang forever; service stops responding | Thread pool saturation, goroutine counts | Timeouts on every call; lint rules |
| **Retry storm** | Traffic 3–10× normal during a dependency issue | Requests ≫ user count; retry metrics | Backoff + jitter, single retry layer, budgets |
| **Breaker never closes / flaps** | Feature permanently off, or on/off | Breaker state metrics | Tune thresholds and half-open probes |
| **Timeout inversion** (outer < inner) | 504s while inner work continues | Mismatch in configs | Timeout budget per call chain, documented |
| **Fallback path broken** | Degradation fails when needed | Never exercised | Test fallbacks (chaos experiments, topic 44) |
| **Health checks fail under load** | LB removes healthy-but-busy instances → cascade | HealthyHostCount drops under load | Separate health endpoint threads, shallow checks |
| **Metastable state** | Outage continues after root cause fixed | Load stays high, success stays low | Shed load at edge, disable retries, then recover gradually |

**3 AM:** "Everything is timing out; the DB is fine now but errors continue." Suspect a metastable retry loop. Actions: block or rate-limit at the edge (shed 50% of traffic), disable client retries via a feature flag if available, let queues drain, then restore traffic gradually.

---

## Where you'll use it

- **Service mesh / gateway configs:** timeouts, retries, outlier detection (topic 10).
- **Kubernetes:** readiness vs liveness (don't kill pods because a dependency is slow), PDBs, resource limits as bulkheads ([K8s 41](../../CKA-Certification-Course-2025/notes/41-requests-limits-qos-quotas.md)).
- **CI/CD:** retries in pipelines for flaky network steps (with backoff), timeouts on jobs ([GHA 14 · job control](../../GitHub-Actions-Basics-To-Production/notes/14-job-control-options.md)).
- **Ansible:** `until`/`retries`/`delay` = retry with backoff ([Ansible 28](../../Ansible/notes/28-loops.md)); `max_fail_percentage` = a circuit breaker for rollouts ([Ansible 31](../../Ansible/notes/31-error-handling.md)).
- **Terraform providers** retry API throttling with backoff internally.

---

## Design exercise

**Prompt:** A product page calls: Catalog (critical), Pricing (critical), Inventory (important), Reviews (nice), Recommendations (nice). Recommendations had an incident where its latency went to 30 s and took the product page down. Design the resilience.

<details>
<summary><b>Model answer</b></summary>

- **Parallel calls** to all five, with a page **deadline** of 800 ms.
- **Timeouts** (from measured p99.9 × ~1.5): Catalog 300 ms, Pricing 200 ms, Inventory 200 ms, Reviews 250 ms, Recommendations 250 ms. Connect timeout 100 ms (same region).
- **Retries:** one retry with jittered backoff for Catalog and Pricing (idempotent GETs), only if the deadline allows; no retries for nice-to-haves. Retry budget 10%.
- **Circuit breakers** per dependency (open at 50% failures over 20 calls; half-open after 15 s).
- **Bulkheads:** separate client connection pools/concurrency limits per dependency (e.g. Recommendations max 50 in-flight per pod) so it can't exhaust shared resources.
- **Fallbacks:** Catalog: serve from cache (stale OK up to 1 h); Pricing: no fallback → if down, show "price unavailable, can't add to cart" (never show a stale price at checkout); Inventory: "check availability at checkout"; Reviews: hide; Recommendations: static "popular items".
- **Load shedding:** concurrency limit on the page service; return a lightweight cached page under overload.
- **Observability:** per-dependency latency/error/breaker state; alert on fallback rate.
- **Test:** chaos experiment injecting 30 s latency into Recommendations in staging, then a small production experiment (topic 44).
</details>

---

## Quick recap

- Every call: timeout (from data), outer > inner, propagate deadlines.
- Retries: only transient + idempotent, exponential backoff with jitter, one layer, retry budget.
- Circuit breakers fail fast and give dependencies room; bulkheads isolate resource pools.
- Shed load early under overload; prioritize critical traffic; bounded queues.
- Plan and test degradation; beware metastable retry loops that outlive their trigger.

### Interview questions

1. **Q: Why add jitter to retries?**
   A: Without jitter, all clients that failed together retry at the same moments (1 s, 2 s, 4 s), creating synchronized spikes that can re-break a recovering service. Random jitter spreads retries over time so load arrives smoothly.

2. **Q: How do you choose a timeout value?**
   A: From the dependency's observed latency distribution: a bit above its p99/p99.9 under normal load, within the caller's own latency budget, and smaller than the caller's caller's timeout minus any retry time. Then I review it when latency profiles change. Never "no timeout", and not arbitrary 30-second defaults.

3. **Q: A slow downstream service caused your whole API to go down. What would you change?**
   A: Tight timeouts on that call, a circuit breaker, a bulkhead (separate pool/concurrency limit) so it can't consume all threads, a fallback or degraded response, shallow health checks so instances aren't removed for being busy, retry budgets to avoid amplification, and load shedding at the front. Then a chaos test to prove it.
