# 34 · Deployment Strategies

> **Level:** ★★★ advanced
>
> **You will learn:** the deployment strategies (**recreate, rolling, blue/green, canary, shadow/dark launch, A/B**), how each works step by step on Kubernetes and cloud load balancers, **feature flags** to separate **deploy** from **release**, **progressive delivery** with **automated canary analysis** and **auto-rollback** (Argo Rollouts, Flagger), **blast radius** control (by %, AZ, region, cell, customer tier), **graceful shutdown** for zero 5xx, handling **stateful** and **long-lived connection** services, and choosing a strategy with trade-offs.
>
> **You need:** topics 09, 12, 23, 28, 33. [K8s 25 · Deployments](../../CKA-Certification-Course-2025/notes/25-deployments.md) covers rolling updates hands-on.
>
> **Part 6 · Delivery** · Previous: [33 · Releases and GitOps](33-releases-promotion-gitops.md) · Next: [35 · Database changes without downtime](35-zero-downtime-database-changes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Most outages are caused by **changes**. Deployment strategies limit how many users a bad change can hurt and how fast you can undo it.

```
Recreate:   stop all v1 → start v2              (downtime; simplest)
Rolling:    replace pods a few at a time         (default K8s; v1 and v2 run together)
Blue/green: run full v2 next to v1, switch all traffic at once, keep v1 for instant switch-back
Canary:     send 1% → 5% → 25% → 100% to v2, checking metrics at each step; auto-rollback on bad metrics
Feature flag: deploy code dark, turn the feature on for 1% of users, then more
```

**Picture it.** Opening a new restaurant menu. Recreate = close the restaurant for a day. Rolling = change one table's menu at a time. Blue/green = build an identical second dining room with the new menu, then move all guests in; if they hate it, move them back. Canary = give the new menu to 1 table in 100, watch their faces, then to more tables.

**The one thing to know.** **Canary + automated analysis on SLIs + automatic rollback** is the senior default for important services. It turns "a bad deploy is an outage" into "a bad deploy affects 1% of requests for 5 minutes".

**Try it now:** `kubectl rollout status deploy/orders` and `kubectl rollout history deploy/orders`.

**You've got it if you can:** compare the strategies in a table, design a canary with analysis criteria, and explain how feature flags separate deploy from release.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Strategy | Mixed versions at once? | Extra capacity | Rollback speed | Blast radius |
|---|---|---|---|---|
| **Recreate** | No | None | Slow (redeploy) | 100% + downtime |
| **Rolling** | Yes | Small (surge) | Medium (roll back = another rolling update) | Grows as rollout proceeds |
| **Blue/green** | No (switch is atomic) | 2× during deploy | **Instant** (switch back) | 100% at switch, but short |
| **Canary** | Yes (small %) | Small | Fast (shift traffic back) | **Small** % |
| **Shadow / dark launch** | v2 gets a copy of traffic, responses discarded | Extra for shadow | N/A (users never see v2) | None (careful with side effects) |
| **A/B testing** | Yes, by user segment | Small | Fast | Segment; goal is product metrics, not safety |
| **Feature flags** | Code for both paths deployed | None | **Instant** (flip flag) | Chosen % / segment |

**Deploy ≠ release:** *deploy* = new code running in production; *release* = users get the new behaviour. Feature flags let you deploy dark and release gradually, independently of deploys.

**Progressive delivery** = canary/flags + automated metric checks + automated promotion/rollback.

---

## Why it exists

Testing can't catch everything: production traffic, data and scale differ. The goal is to **detect bad changes with minimal exposure** and **undo them fast**. DORA's "change failure rate" and "time to restore" (topic 39) are directly improved by these strategies.

---

## Analogy

Restaurant menu (Core). **Feature flag** = the new dish is already in the kitchen, but only waiters with a special card may offer it; the manager can take the card away instantly.

---

## How it works inside

### Rolling update (Kubernetes Deployment)

```
replicas: 10, maxSurge: 25% (=3), maxUnavailable: 0
step: create up to 3 v2 pods → wait until Ready (readiness probe) → terminate 3 v1 pods → repeat
v1 ██████████          v1 ███████   v2 ███          ...          v2 ██████████
```
- `maxUnavailable: 0` keeps full capacity; `maxSurge` controls speed and extra cost.
- **Readiness probe** gates progress; `minReadySeconds` waits a bit after ready; `progressDeadlineSeconds` marks a stuck rollout as failed (but doesn't auto-rollback by itself).
- Both versions serve at once → they must be **compatible** (API, DB schema, message formats).

### Graceful shutdown (needed by every strategy for zero 5xx)

```
1. Pod marked Terminating → removed from EndpointSlices (async) ; preStop hook runs
2. preStop: sleep 10–15 s  (lets kube-proxy/LB/mesh stop sending new traffic)
3. SIGTERM → app stops accepting new connections, finishes in-flight requests, closes pools
4. terminationGracePeriodSeconds (e.g. 45 s) → SIGKILL if still running
LB side: deregistration delay ≥ request duration (topic 09)
```

### Blue/green

```
           ┌──▶ BLUE  (v1, live)  ◀── 100% traffic
[LB/DNS/Ingress]
           └──▶ GREEN (v2, idle, fully deployed, tested with internal traffic)
Switch: point listener / Service selector / DNS weight to GREEN → 100% to v2 at once
Rollback: point back to BLUE (seconds). Tear down BLUE after confidence period.
```
- Switch mechanisms: LB target group swap (ALB listener rule), K8s Service selector change, DNS weighted records (slow, TTL), Argo Rollouts `blueGreen`.
- Pros: instant switch/rollback, full pre-production test of v2 in prod environment.
- Cons: 2× capacity during the deploy; all users switch at once (a bug hits 100%, though briefly); DB is shared → schema must be compatible with both; long-lived connections/sessions need draining.

### Canary with automated analysis

```
step 1: 5% traffic → v2, pause 5 min, analysis
step 2: 25%, pause 10 min, analysis
step 3: 50%, pause 10 min, analysis
step 4: 100%
Analysis (each step): compare v2 vs v1 (baseline) on
   - error rate (5xx ratio)            fail if v2 > v1 + 0.5% or > SLO threshold
   - latency p99                       fail if v2 > 1.2 × v1
   - saturation / pod restarts         fail if crashloops
   - business metric (checkout success)
If any check fails → abort: traffic back to 100% v1, alert team
```
- **Traffic splitting** needs an L7 router: service mesh (Istio, Linkerd), ingress/gateway (NGINX, Gateway API weights), ALB weighted target groups, or replica-count-based (coarse: 1 of 10 pods ≈ 10%).
- **Tools:** Argo Rollouts (`Rollout` CRD with steps + `AnalysisTemplate` querying Prometheus), Flagger, Spinnaker/Kayenta, AWS CodeDeploy (Lambda/ECS canary), cloud-native options.
- **Statistical caution:** at 1% of low traffic, there may be too few requests to judge. Use a minimum request count per step, longer steps, or synthetic traffic.
- **Sticky canary:** route a given user consistently to v2 (header/cookie hash) so they don't flip between versions.

### Feature flags

```
if flags.enabled("new-checkout", user=ctx.user):   # evaluated locally from cached rules
    new_checkout()
else:
    old_checkout()
```
- Release to internal staff → 1% → 10% → 50% → 100%; kill switch instantly.
- Targeting by user, tenant, region, plan.
- **Flag debt:** remove flags after full release; old flags = dead code paths and confusing combinations.
- Flag service must be fast and resilient (local evaluation, cached rules, safe defaults): case study 57.

### Blast radius: other dimensions to roll out by

```
by % of traffic → by AZ → by region → by cell → by customer tier (internal → free → paid → enterprise)
"Wave" deployments: deploy to 1 cell/region, bake for hours, then the next wave
```
Large providers deploy region by region with **bake times**, because some bugs only appear after hours (memory leaks, cron-based code, daily batch). Config changes deserve the same progressive rollout as code.

### Special cases

- **Long-lived connections (WebSockets, gRPC streams):** rolling/canary only affects new connections; old connections stay on v1 until they end. Use connection max-age, server-initiated graceful close with client reconnect + jitter.
- **Stateful services (DBs, Kafka, Elasticsearch):** rolling restarts one node at a time with health checks between (StatefulSet `RollingUpdate`, `partition` for canary-style), respecting replication/quorum (topic 26). Never restart two quorum members at once.
- **Workers/consumers:** canary by running a small v2 consumer group share; watch DLQ rates.
- **Mobile/desktop clients:** staged rollout in app stores + server-side flags; old versions live for months → backward-compatible APIs (topic 06).
- **Infrastructure changes (Terraform):** "canary" by environment and by small blast-radius states first (topic 36).

---

## Options and trade-offs

| Need | Choose |
|---|---|
| Dev/test, or app can't run two versions at once (exclusive file lock, singleton) | Recreate |
| Default stateless service, moderate risk | Rolling + good probes + graceful shutdown |
| Instant rollback, full pre-switch verification, capacity is affordable | Blue/green |
| Critical, high-traffic service | **Canary with automated analysis** |
| Risky new feature or product experiment | Feature flags (+ canary for the deploy itself) |
| Rewrite / new backend you can't expose yet | Shadow traffic (careful: no duplicate side effects; write paths need stubs) |

| Cost / complexity | Low → High |
|---|---|
| Recreate → Rolling → Blue/green (capacity) → Canary with mesh + analysis (tooling) → Cells/waves (architecture) |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **No graceful shutdown** | 502/503 spike every deploy | 5xx aligned with deploys | preStop sleep, SIGTERM handling, deregistration delay |
| **Readiness probe too shallow** | Broken v2 pods receive traffic | Errors from new pods | Real readiness (config loaded, warm), `minReadySeconds` |
| **Version incompatibility during rollout** | Errors only while mixed | Error spike mid-rollout | Backward/forward-compatible APIs, schemas, messages (topic 35) |
| **Canary analysis too weak** | Bad version promoted | Post-rollout incidents | Compare to baseline, include business metrics, minimum sample size |
| **Canary too small to detect** | Noise, flapping decisions | Low request counts | Longer steps, synthetic traffic, higher first step |
| **Bug appears hours later** | Incident after 100% rollout | Late errors | Bake times, waves by region/cell |
| **Blue/green DB divergence** | Rollback to blue fails because data changed shape | Errors on switch-back | Shared DB with compatible schema; expand/contract |
| **Flag combination explosion** | Hard-to-reproduce bugs | — | Flag ownership, expiry, cleanup |
| **Rollback broken** (old image deleted, migration irreversible) | Can't undo | Rollback drill | Keep N previous images, test rollback regularly |

**3 AM:** "Canary auto-aborted, team paged." Good: the system worked; only a small % saw errors. Check the analysis run to see which metric failed, look at v2 logs/traces, fix forward in working hours. If the canary didn't abort and errors are high: abort the rollout manually and roll back (topic 30).

---

## Where you'll use it

- **Case study 48** (zero-downtime deployment platform) puts all of this together.
- **Argo Rollouts / Flagger** on K8s; **ALB weighted target groups** via Terraform for VMs/ECS.
- **Ansible rolling deploys** with `serial`, LB drain and `max_fail_percentage` ([Ansible 49](../../Ansible/notes/49-project-rolling-deploy.md)).
- **Jenkins/GHA** deploy stages with approvals and environment protection.
- **Interviews:** "How do you deploy with zero downtime?" is a top-10 DevOps question.

---

## Design exercise

**Prompt:** A payments API (2,000 req/s, SLO 99.95%) on EKS deploys 10 times a day with rolling updates. Twice last quarter, a bad deploy caused 15-minute outages before someone rolled back. Design the new deployment strategy.

<details>
<summary><b>Model answer</b></summary>

- **Argo Rollouts canary** with traffic splitting via the mesh/Gateway API: steps 2% (5 min) → 10% (10 min) → 50% (10 min) → 100%.
- **AnalysisTemplate** (Prometheus), comparing canary vs stable at each step: 5xx ratio (fail if canary > stable + 0.2 percentage points), p99 latency (fail if > 1.25× stable), payment authorization success rate (business metric), pod restarts = 0. Minimum 2,000 requests per step (2% of 2,000 req/s × 300 s = 12,000 → fine).
- **Auto-abort** → traffic to stable instantly, notify the team; the Git commit is marked failed.
- **Graceful shutdown** settings: preStop 10 s, grace 45 s, readiness with `minReadySeconds: 10`.
- **Compatibility rule:** every release must work alongside the previous one (API, schema via expand/contract (topic 35), events).
- **Feature flags** for risky business logic (new fraud rule) → released separately from deploys.
- **Error budget gate:** promotion to prod blocked if < 25% budget left (except fixes).
- **Waves:** if multi-region, region 1 first with 1-hour bake, then region 2.
- **Expected result:** a bad deploy affects ~2% of requests for ~5 minutes (≈ 2,000 × 0.02 × 300 = 12,000 requests worst case, ~0.02% of monthly traffic), instead of 100% for 15 minutes; MTTR becomes automatic.
</details>

---

## Quick recap

- Recreate (downtime), rolling (default), blue/green (instant switch, 2× capacity), canary (small blast radius), shadow (no user impact), flags (decouple release from deploy).
- Canary + automated analysis vs baseline + auto-rollback = senior default for critical services.
- Zero 5xx needs readiness gating, preStop delay, SIGTERM handling and LB draining.
- Mixed versions run during rollouts: keep APIs, schemas and events compatible.
- Roll out by %, AZ, region, cell, customer tier, with bake time; treat config changes the same way.

### Interview questions

1. **Q: Blue/green or canary?**
   A: Blue/green gives an instant, all-at-once switch and switch-back with full pre-switch testing, at 2× capacity during deploys, but every user hits a bug at the switch. Canary exposes only a small percentage first and uses metrics to decide, which limits blast radius but needs L7 traffic splitting, analysis and tolerance of mixed versions. For high-traffic critical services I prefer canary with automated analysis; blue/green suits apps where mixed versions are a problem or traffic is too low for meaningful canary stats.

2. **Q: What makes a rolling update truly zero-downtime on Kubernetes?**
   A: `maxUnavailable: 0`, accurate readiness probes, a preStop delay so endpoints/LBs stop routing before shutdown, the app handling SIGTERM by draining in-flight requests, a sufficient termination grace period, LB deregistration delay, PDBs, and version compatibility between old and new pods (APIs and schema).

3. **Q: How do feature flags change your deployment strategy?**
   A: They separate deploy from release: code ships dark through the normal pipeline, and the feature is enabled gradually by user segment with an instant kill switch, without redeploying. That makes deploys smaller and less risky, but you must manage flag debt, test flag combinations, and make the flag system itself highly available with safe defaults.
