# 31 · Capacity Planning and Performance Testing

> **Level:** ★★★ advanced
>
> **You will learn:** what **capacity planning** is (organic growth, inorganic events, headroom), **Little's Law** and **queueing theory** in plain words (why latency explodes near 100% utilization), how to find **per-instance capacity** with load tests, the test types (**load, stress, spike, soak, breakpoint**), designing a realistic **load test** (workload model, test data, environment), **forecasting** from metrics, **headroom and N+1 per AZ**, **cloud quotas**, and preparing for **big events** (sales, launches).
>
> **You need:** topics 03, 11, 12, 23, 27.
>
> **Part 5 · Observability and SRE** · Previous: [30 · Incident management](30-incident-management.md) · Next: [32 · CI/CD platform design](32-cicd-platform-design.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Capacity planning** = making sure you'll have enough resources (compute, DB, network, quotas, money) **before** demand arrives, with safe headroom, at reasonable cost.

```
Little's Law:      L = λ × W          (items in system = arrival rate × time each spends)
                   500 req/s × 0.2 s = 100 requests in flight → need ≥ 100 concurrent workers/threads

Utilization (ρ) vs waiting:  wait grows like ρ / (1 − ρ)
   50% busy → wait ≈ 1× service time      80% → 4×      90% → 9×      95% → 19×
```

**Picture it.** A supermarket. Little's Law: if 2 customers arrive per minute and each spends 15 minutes inside, there are ~30 customers in the shop. Queueing: when cashiers are 95% busy, any small burst creates long queues; at 70% busy, queues stay short.

**The one thing to know.** **Don't run hot.** Plan for peak at ~60–70% utilization of the bottleneck, survive losing one AZ, and know the **breaking point** from a load test, not from hope.

**Try it now:**
```bash
# k6 (load testing tool): 50 virtual users for 2 minutes against a test endpoint
k6 run --vus 50 --duration 2m script.js
```

**You've got it if you can:** apply Little's Law, explain why latency explodes near 100%, plan a load test, and size a fleet for a sale event.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **Organic growth:** steady user/traffic growth (e.g. 5% month over month).
- **Inorganic demand:** launches, marketing campaigns, sales (Black Friday, Diwali), customer onboarding, migrations.
- **Headroom:** spare capacity above expected peak (buffer for spikes, failures, forecast errors).
- **Capacity model:** the relationship "X units of resource serve Y units of demand", e.g. "1 pod (2 vCPU) = 250 req/s at p99 < 200 ms".
- **Bottleneck resource:** the one that saturates first (topic 11).
- **Quota / service limit:** cloud-imposed caps (vCPUs per region, EIPs, ENIs, API rate limits) that block scaling no matter how much money you have.

---

## Why it exists

Autoscaling (topic 12) adjusts within limits; it can't fix: a database that can't grow fast, cloud quotas, reserved capacity commitments, physical lead times, or the fact that new capacity takes minutes. Capacity incidents are among the most common outages. Over-provisioning, on the other side, wastes money (topic 43). Planning balances both.

---

## Analogy

The supermarket (Core). Extra: a **stadium** planning for a final. They know ticket numbers (forecast), entrance throughput per gate (capacity model), and that gates jam above ~80% throughput (queueing). They open extra gates and hire staff **before** the day (pre-scaling), and rehearse (load test).

---

## How it works inside

### Little's Law in practice

`L = λ × W` holds for any stable system, regardless of distribution.

| Question | Use |
|---|---|
| How many threads/connections do I need? | 2,000 req/s × 50 ms DB time = 100 concurrent DB connections in use |
| How many runners? | 12.5 jobs/min × 10 min = 125 (topic 03) |
| How big will the queue get? | 300 msg/s × 40 s processing delay = 12,000 messages in flight |
| What happens if latency doubles? | Same traffic, W doubles → concurrency doubles → pools may exhaust → cascading failure (topic 24) |

That last row is key: **slower dependencies need more concurrency**, which is why a slow DB exhausts app thread pools.

### Queueing: why not run at 95%?

For a simple single-server queue with random arrivals, the average wait is roughly `service_time × ρ / (1 − ρ)`:

```
ρ (utilization)   wait (× service time)
   0.50              1
   0.70              2.3
   0.80              4
   0.90              9
   0.95             19
   0.99             99
```
Real systems with many servers do better, but the shape is the same: **latency curves are hockey sticks**. Small traffic increases near saturation cause huge latency increases. Targets of 60–70% at peak keep you on the flat part.

### Finding per-instance capacity (the capacity model)

```
1. Deploy ONE instance (or a fixed small set) in a prod-like environment
2. Ramp load in steps (e.g. +50 req/s every 2 min) with a realistic request mix
3. Record throughput, p50/p99 latency, error rate, CPU, memory, GC, DB time
4. Find the "knee": where p99 crosses your SLO threshold or errors start
5. Capacity per instance = ~70–80% of the knee throughput (safety)
6. Repeat after significant code or infra changes (in CI for critical services)
```

### Test types

| Type | Shape | Finds |
|---|---|---|
| **Load test** | Expected peak, sustained | Can we meet SLOs at expected load? |
| **Stress test** | Beyond peak until it breaks | Breaking point and **how** it fails (gracefully?) |
| **Spike test** | Sudden jump (10× in 1 min) | Autoscaling lag, cold caches, connection storms |
| **Soak / endurance** | Normal load for hours/days | Memory leaks, connection leaks, disk growth, log rotation |
| **Breakpoint / capacity test** | Slow ramp until SLO breach | Max capacity number |
| **Failover-under-load** | Load + kill an AZ/DB primary | N+1 really works? Retry storms? (topic 44) |

### Designing a realistic load test

- **Workload model:** the real mix of endpoints (from access logs/traces), think times, user sessions, read:write ratio, payload sizes.
- **Test data:** realistic DB size and distribution (a 1,000-row DB in staging makes every query fast); avoid all virtual users hitting the same product (unrealistic cache hit rate) or all unique (unrealistic misses).
- **Environment:** prod-like (same instance types, same config, same DB class). Or test **in production** carefully: off-peak, with synthetic tenants, gradual ramp, abort criteria, and stakeholder sign-off.
- **Load generators:** must not be the bottleneck (distribute k6/Locust/Gatling/JMeter workers; check their CPU).
- **Measure from the user side** (generator latency) **and** inside (server metrics, traces).
- **Abort criteria:** stop if error rate > X% or if production impact appears.
- **Dependencies:** stub third parties (don't load-test a payment provider without permission!) or use their sandbox with agreed limits.

### Forecasting

```
1. Take the peak (e.g. daily p95 of hourly peak RPS) for the last 6–12 months
2. Fit a trend (linear / growth rate); overlay seasonality (weekly, yearly events)
3. Add known inorganic events (marketing: "+3× on Nov 28")
4. Convert demand → resources using the capacity model, per bottleneck resource
5. Compare with current capacity, quotas, commitments → plan purchases/changes with lead time
```
PromQL helpers: `predict_linear()` for disks and simple trends. Review forecasts monthly; capacity plans quarterly.

### Sizing rules

```
Required instances = peak RPS / (capacity per instance)
Spread over AZs with N+1:  per_AZ = required / (AZs − 1)
Total = per_AZ × AZs

Example: peak 6,000 req/s, 250 req/s per pod (at 70% of knee) → 24 pods
3 AZs, survive one AZ loss → per AZ = 24 / 2 = 12 → total 36 pods at peak
```

Also check every **shared** resource: DB connections (36 pods × pool size), DB CPU at peak, cache memory/ops, queue throughput, NAT gateway bandwidth/ports, load balancer capacity units, **cloud quotas** (vCPU limits per instance family per region, IP addresses in subnets, EBS volumes), third-party API limits.

### Preparing for a big event (checklist)

1. Forecast peak with business (marketing plan, past events).
2. Load test to 1.5–2× forecast; fix bottlenecks found.
3. Raise quotas and limits weeks ahead; reserve capacity (On-Demand Capacity Reservations) if needed.
4. Pre-scale (min replicas, DB instance size, cache cluster), pre-warm caches/CDN, notify cloud provider/CDN for very large events.
5. Code/config freeze around the event; on-call staffed, war room, runbooks ready.
6. Load shedding and degradation plan ready (topic 24); queue for non-critical work.
7. After: scale down, review real vs forecast.

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Headroom | Generous (target 50%): safe, costs more | Tight (80%): cheap, risky in spikes and AZ loss |
| Test environment | Prod-like staging: safe, expensive to keep identical | Testing in prod: realistic, needs careful controls |
| Capacity source | Autoscaling only | Autoscaling + pre-scaling + reservations for known peaks ✅ |
| Commitments (topic 43) | Savings Plans/Reserved: cheaper, locked in | On-demand: flexible, pricier |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Running hot** | Latency spikes at every small burst | Utilization at peak > 80% | Headroom targets per resource |
| **Unknown bottleneck** | Scaling app doesn't help | Load test, USE dashboards | Capacity model per resource |
| **Quota hit during scale-out** | ASG/Karpenter can't launch | Launch errors | Quota monitoring and requests in advance |
| **Unrealistic load test** | Passed test, failed in prod | Prod ≠ test metrics | Real traffic mix, data volume, cache behaviour |
| **Soak issues** | Fails after days (leaks) | Memory slowly rising | Soak tests, leak alerts |
| **Event forecast missed** | Outage at launch | — | Business involvement, 2× margin, load shedding |
| **Load generator bottleneck** | Flat throughput, low server CPU | Generator CPU/network | Distributed generators |

**3 AM:** an unexpected viral spike. Mitigate with what's ready: raise HPA max/min and node limits, shed non-critical traffic, enable static/cached fallbacks, rate limit abusive clients, scale DB vertically if you must (failover blip). Then plan properly.

---

## Where you'll use it

- **Design interviews:** "How many servers?" → capacity model + N+1 per AZ math.
- **K8s:** resource requests from measured usage (VPA recommendations), cluster sizing, node pool limits ([K8s 49](../../CKA-Certification-Course-2025/notes/49-vpa-inplace-resize-node-autoscaling.md)).
- **CI/CD:** performance tests as a pipeline stage for critical services (topic 32).
- **CI runner fleets:** Little's Law for runner counts (topic 47).
- **Databases and disks:** `predict_linear` growth alerts; storage planning.

---

## Design exercise

**Prompt:** An e-commerce API averages 2,000 req/s, normal daily peak 5,000 req/s. Marketing expects a sale to bring 4× normal peak for 2 hours. Load tests show one pod's knee at 400 req/s (p99 crosses 300 ms). Each pod uses a DB pool of 10; Aurora primary handles 30,000 queries/s at ~70% CPU; each request makes ~3 queries, 70% served by cache on normal days. Plan capacity.

<details>
<summary><b>Model answer</b></summary>

- **Forecast:** 4 × 5,000 = **20,000 req/s**; plan for **1.5×** uncertainty → 30,000 req/s design target (or at least load test to it).
- **Pods:** capacity per pod = 0.75 × 400 = 300 req/s → 30,000 / 300 = 100 pods. N+1 over 3 AZs → 100 / 2 = 50 per AZ → **150 pods**. Pre-scale `minReplicas` to ~120 an hour before; HPA max 200. Nodes: pre-provision via Karpenter/overprovisioning; check vCPU quotas (150 pods × 2 vCPU = 300 vCPU + system overhead).
- **DB connections:** 150 × 10 = 1,500 connections → too many for comfort → RDS Proxy/PgBouncer, or reduce pool to 4–5 per pod (Little's Law: queries 30,000 req/s × 3 × 0.3 miss = 27,000 queries/s × ~2 ms = ~54 connections busy on average; pools of 5 are plenty).
- **DB load:** 30,000 req/s × 3 queries × 30% miss = **27,000 queries/s** ≈ the primary's 70% mark (30,000). Too close at the 1.5× target. Actions: raise cache hit ratio for sale items (pre-warm cache for sale catalog → 90% hit → 9,000 queries/s), move read-heavy endpoints to Aurora replicas, temporarily scale the writer up one size.
- **Cache:** check Redis ops/s (30,000 × 3 = 90,000 ops/s) → 3+ shards; memory for sale catalog.
- **Other:** ALB pre-warming not usually needed for gradual ramps, but contact AWS for sudden huge spikes; CDN for product images; NAT and third-party limits (payment provider: notify them).
- **Load test** the whole stack at 30,000 req/s two weeks before; soak at 20,000 for 2 h; failover test (kill one AZ's pods) under load.
- **Plan B:** queue-based checkout waiting room if load exceeds plan; non-critical features (recommendations) disabled via flags during the peak.
</details>

---

## Quick recap

- Plan for organic growth and inorganic events, with headroom and N+1 per AZ.
- Little's Law (L = λW) sizes concurrency: threads, connections, runners, queues.
- Queueing: latency explodes as utilization → 100%; target ~60–70% at peak.
- Load, stress, spike, soak and failover-under-load tests; realistic mix, data and environment.
- Check every shared resource and cloud quotas; pre-scale and rehearse for big events.

### Interview questions

1. **Q: Why not run servers at 90% CPU to save money?**
   A: Queueing delay grows sharply near saturation (wait ∝ ρ/(1−ρ)): at 90% a small burst makes queues and latency explode, retries add more load, and there's no room to absorb an AZ failure or autoscaling lag. Running ~60–70% at peak trades some cost for stable latency and failure tolerance; cheaper capacity (Spot, right-sizing) is a better saving lever.

2. **Q: How do you determine how many pods a service needs for an event?**
   A: Load test one pod with a realistic request mix to find the knee where SLOs break, take ~70–80% of it as per-pod capacity, divide forecast peak (with margin) by it, then add N+1 across AZs. Then check shared resources: DB, cache, connections, quotas, third parties, which usually bind first.

3. **Q: Explain Little's Law with an infrastructure example.**
   A: Items in a system equal arrival rate times time in system. If 2,000 requests/s each hold a DB connection for 50 ms, about 100 connections are in use. If DB latency doubles to 100 ms, you need 200, and if the pool caps at 150, requests queue, latency rises further, and it cascades. That's why pools and timeouts are sized from Little's Law.
