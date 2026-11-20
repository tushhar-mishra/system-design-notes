# 23 · Availability and the Nines

> **Level:** ★★ intermediate
>
> **You will learn:** what **availability** means and how to measure it, the **nines** table and quick math, **serial vs parallel** availability (why dependencies multiply risk and redundancy helps), **MTBF/MTTR** and why **fast recovery beats rare failure**, **failure domains** (process → host → AZ → region → provider), finding **SPOFs** (Single Points Of Failure), **N+1 / N+2** redundancy, **correlated failures**, **static stability**, and the **cost curve** of each extra nine.
>
> **You need:** topics 03, 08, 09, 17.
>
> **Part 4 · Reliability foundations** · Previous: [22 · Async patterns](22-async-patterns.md) · Next: [24 · Resilience patterns](24-resilience-patterns.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Availability** = the fraction of time (or of requests) the system works for users.

```
99%     → 3.65 days/year down     → 7.2 h/month
99.9%   → 8.76 h/year             → 43.8 min/month
99.95%  → 4.38 h/year             → 21.9 min/month
99.99%  → 52.6 min/year           → 4.4 min/month
99.999% → 5.26 min/year           → 26 s/month
```

**Two rules:**
```
In SERIES (A needs B needs C):  A_total = A × B × C          → 0.999 × 0.999 × 0.999 = 0.997 (worse!)
In PARALLEL (any of N works):   A_total = 1 − (1 − A)^N      → 1 − (0.01)^2 = 0.9999 for two 99% copies
```

**Picture it.** Christmas lights. Old-style strings in **series**: one bulb fails, the whole string goes dark; more bulbs = more likely to be dark. Modern strings in **parallel**: one bulb fails, the rest stay lit.

**The one thing to know.** Every hard dependency **lowers** your availability; every redundant copy **raises** it, *if failures are independent*. Shared things (same AZ, same config push, same bug) make copies fail together.

**Try it now:** list the hard dependencies of one request in your app (DNS, LB, app, DB, cache, auth service, third-party API). Multiply their availabilities.

**You've got it if you can:** convert nines to minutes, compute serial and parallel availability, and find the SPOFs in a diagram.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**Two ways to measure:**
- **Time-based:** uptime ÷ total time. Simple, but "up" is fuzzy (is it up if 30% of requests fail?).
- **Request-based** ✅: good requests ÷ total requests. Matches user experience; this is what SLOs use (topic 28).

**Key terms:**
- **MTBF** (Mean Time Between Failures): how often things break.
- **MTTR** (Mean Time To Recovery/Repair): how long it takes to recover. Often split into **MTTD** (detect) + time to mitigate.
- `Availability ≈ MTBF / (MTBF + MTTR)`
- **SPOF** (Single Point Of Failure): a component whose failure alone takes the system down.
- **Failure domain:** a set of things that fail together (a host, a rack, an AZ, a region, a cloud provider, a config system, a team's deploy).
- **Blast radius:** how much is affected when one failure domain fails.
- **N+1:** enough capacity to serve peak with one unit (instance/AZ) lost. **N+2:** survive two (e.g. one in maintenance + one failing).

---

## Why it exists

Availability targets drive architecture and cost. "99.9%" is achievable in one region with multi-AZ and good practices. "99.99%" needs automated failover everywhere, fast detection and deployment safety. "99.999%" typically needs multi-region active-active and a very mature organisation. Each nine costs roughly **an order of magnitude more effort**. Designing to the right number (not the biggest) is a senior skill.

---

## Analogy

Christmas lights (Core). Extras:
- **Correlated failure** = all bulbs on the same faulty transformer: parallel wiring doesn't help if the plug fails.
- **MTTR** = how fast you can find and replace the dead bulb. A string you can fix in 10 seconds is "more available" than one that rarely fails but takes a day to fix.

---

## How it works inside

### Quick math

```
Downtime allowed per year = (1 − A) × 525,600 minutes
99.95%: 0.0005 × 525,600 ≈ 263 min ≈ 4.4 h
Per 30-day month = (1 − A) × 43,200 minutes
99.9%:  0.001 × 43,200 = 43.2 min
```

### Serial dependencies (worked)

A request path: DNS (99.99) → CDN (99.99) → LB (99.99) → app (99.95) → DB (99.95) → auth service (99.9) → payment provider (99.9).

```
0.9999^3 × 0.9995^2 × 0.999^2
≈ 0.9997 × 0.9990 × 0.9980 ≈ 0.9967 → 99.67%  ≈ 2.4 h/month of failed time
```
The system is far worse than any part. **Removing or softening dependencies** is the biggest lever:
- Make the payment provider async (queue) or degrade gracefully → it leaves the critical path.
- Cache auth decisions (validate JWTs locally, topic 10) → auth service off the per-request path.

### Parallel redundancy (worked)

```
One app instance: 99%              → 1 − 0.01   = 99%
Two independent:  1 − 0.01²        = 99.99%
Three:            1 − 0.01³        = 99.9999%
```
⚠️ Only if failures are **independent**. Two instances in the same AZ, same host, same bad deploy → they fail together. The real formula is capped by the **shared** parts: LB, config, deploy pipeline, DB.

### Combining: a three-AZ web tier

```
          [LB 99.99%] 
         /    |     \
     AZ-a   AZ-b   AZ-c     each AZ's app tier ≈ 99.9%; survive any one AZ (N+1 capacity)
         \    |     /
     [DB Multi-AZ 99.95%]

App tier (need ≥2 of 3 AZs): ≈ 1 − 3 × 0.001² ≈ 99.9997%
Total ≈ 0.9999 × 0.999997 × 0.9995 ≈ 99.94%  → DB and LB dominate
```
Lesson: once a tier is redundant, it stops being the weak link; the **next SPOF** dominates. Keep walking the path.

### MTTR beats MTBF

```
System A: fails once a year, takes 8 h to recover   → 8 h/year down → 99.91%
System B: fails once a month, recovers in 2 min     → 24 min/year   → 99.995%
```
Fast detection (good alerts, topic 29), automated failover, fast rollback (topic 34), and practised runbooks shorten MTTR. Most of SRE work is reducing MTTR and blast radius, not achieving zero failures.

### Failure domains and blast radius

```
Process < Container/Pod < Node/VM < Rack < AZ < Region < Cloud provider
                                                         + "logical" domains:
   a config push, a DNS change, a certificate, a deploy, an IAM change, a shared library, one team
```
The biggest real outages often come from **logical** failure domains: a bad config pushed everywhere at once. Physical redundancy doesn't help; **progressive rollout** of every change does (topic 34). **Cell-based architecture** (topic 42) shrinks blast radius by splitting users into independent copies.

### Static stability

A system is **statically stable** if it keeps working during a failure **without needing to make changes** (no new instances, no control-plane calls). Example: running at 3 AZs with enough capacity in each to lose one, rather than relying on autoscaling to launch replacements during an AZ outage (when the control plane may be busy or impaired). AWS designs its own services this way.

### Finding SPOFs: the walk

For each box and arrow on your diagram, ask: "If this one thing fails, does the user-facing function fail?"

Common SPOFs:
- A single NAT gateway, single DB without standby, single Redis, single Jenkins controller.
- One DNS provider, one certificate, one CI pipeline that's needed to roll back.
- A cron job on one VM.
- A human: the only person with access/knowledge (bus factor).
- A third-party API on the critical path.

---

## Options and trade-offs

| Target | Typical design | Rough cost vs baseline | Notes |
|---|---|---|---|
| 99% | Single instance, backups | 1× | Internal tools |
| 99.9% | Multi-AZ, LB, managed DB with standby, health checks, on-call | ~1.3–1.5× | Most business apps |
| 99.95% | + automated failover everywhere, progressive deploys, tested restores | ~1.5–2× | Customer-facing core |
| 99.99% | + no manual steps in recovery, cells or regional isolation, dependency isolation, 24/7 mature on-call | ~2–3×+ | Payments, critical APIs |
| 99.999% | Multi-region active-active, consensus data stores, chaos testing, huge engineering | ~5×+ | Very rare; telecom, top cloud services |

**Your availability can't exceed your critical dependencies'** unless you remove them from the path. If your cloud region's managed DB offers a 99.95% SLA (illustrative), a single-region design depending on it can't promise 99.99% without extra architecture.

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Hidden SPOF** | Small failure → total outage | Postmortems, game days (topic 44) | SPOF walk at design review (topic 59) |
| **Correlated failure of "redundant" parts** | All replicas die together | Simultaneous failures | Spread across AZs, staggered deploys, diverse dependencies |
| **Insufficient N+1 capacity** | AZ loss → survivors overload → total outage | Utilization per AZ | Size for N−1 AZs at peak |
| **Failover never tested** | Standby doesn't work when needed | First real incident | Regular switchover drills |
| **Long MTTR** | Small issues become long outages | Incident timelines | Alerts on symptoms, runbooks, one-click rollback |
| **Measuring the wrong availability** | Dashboards green, users angry | Complaints vs metrics | Request-based SLIs measured at the edge (topic 28) |

**3 AM:** an AZ fails. If designed right (N+1 per AZ, statically stable), alarms fire but users see a small blip. If not, survivors overload → cascading failure. Immediate: shed load (topic 24), shift traffic, scale manually where the control plane allows.

---

## Where you'll use it

- **Setting SLOs** (topic 28) and negotiating SLAs.
- **Design interviews:** "What's the availability of your design?" → do the serial/parallel math.
- **K8s:** replicas across zones (topology spread, [K8s 46](../../CKA-Certification-Course-2025/notes/46-pod-affinity-and-topology-spread.md)), PDBs, 3 control-plane nodes ([K8s 78](../../CKA-Certification-Course-2025/notes/78-ha-control-plane-and-managed-kubernetes.md)).
- **CI/CD:** the pipeline is part of your recovery path (rollback needs it) → its availability matters (topic 47).
- **Jenkins HA** discussion ([Jenkins 69](../../Jenkins/notes/69-monitoring-performance-ha.md)).

---

## Design exercise

**Prompt:** The business asks for 99.99% availability for a checkout API. Today: single region, 3 AZs, EKS, Aurora Multi-AZ, ElastiCache, a third-party fraud-check API (99.9% SLA) called synchronously, and a third-party payment provider (99.95%). Can you meet it? What changes?

<details>
<summary><b>Model answer</b></summary>

**Current serial math** (rough): edge/LB 99.99 × EKS app (multi-AZ) ~99.99 × Aurora ~99.99 × cache (if hard dependency) 99.9+ × fraud 99.9 × payment 99.95 ≈ **99.73%**. The third parties alone cap us at ~99.85%. **No**, not as designed.

**Changes:**
1. **Fraud check off the hard path:** strict timeout (e.g. 300 ms); on timeout/failure, apply a fallback rule (allow low-risk orders up to a limit, queue for async review, or route to a second provider). Now fraud failures don't fail checkout.
2. **Payment provider:** add a **second provider** with automatic failover (parallel: 1 − 0.0005 × 0.0005 ≈ 99.99997% if independent), idempotency keys per attempt.
3. **Cache:** make it optional (topic 14): timeouts + fallback to DB with limits.
4. **Deployment safety:** most outages are changes, so canary + automated rollback (topic 34), progressive config rollout.
5. **MTTR:** SLO burn-rate alerts (topic 29), runbooks, regular failover drills.
6. **Region:** 99.99% is possible single-region if every dependency is redundant, but the remaining risk is a regional event. Ask the business: is a region outage in scope? If yes, warm-standby or active-active multi-region (topic 42), roughly 2× cost.

Also negotiate how it's measured: request-based, at the edge, monthly, excluding client errors.
</details>

---

## Quick recap

- Measure availability by good requests ÷ total; know the nines table cold.
- Serial dependencies multiply (worse); parallel redundancy improves only if failures are independent.
- MTTR matters as much as MTBF: detect fast, fail over automatically, roll back fast.
- Failure domains include logical ones (config, deploys); limit blast radius with progressive rollout and cells.
- Each extra nine costs roughly 10× effort; design to the needed target and remove dependencies from the critical path.

### Interview questions

1. **Q: Your service depends on five services, each 99.9%. What's your availability?**
   A: About 0.999^5 ≈ 99.5%, roughly 3.6 hours a month. To do better I'd remove hard dependencies from the critical path (async, caching, fallbacks), add redundancy where the dependency is ours, and use timeouts so a slow dependency degrades a feature instead of the whole request.

2. **Q: How would you get from 99.9% to 99.99%?**
   A: Find what consumed the error budget: usually bad deploys, config changes and dependency failures. Then progressive delivery with automated rollback, eliminate SPOFs, size N+1 per AZ for static stability, isolate dependencies with timeouts/circuit breakers/fallbacks, and cut MTTR with symptom-based alerts and automated failover. Also verify dependencies can support 99.99% at all.

3. **Q: Why might two replicas not give you the availability the formula predicts?**
   A: The formula assumes independent failures. Replicas often share failure domains: same AZ, same node, same config, same release, same database. A bad deploy or config push hits both at once. Spread across failure domains and roll changes out progressively.
