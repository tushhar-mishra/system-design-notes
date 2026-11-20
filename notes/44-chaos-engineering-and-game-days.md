# 44 · Chaos Engineering and Game Days

> **Level:** ★★ intermediate
>
> **You will learn:** what **chaos engineering** is (and isn't), the **experiment method** (steady state → hypothesis → inject fault → observe → learn), **blast-radius control** and **abort conditions**, the common **fault types** (instance/pod/AZ loss, latency, errors, dependency failure, resource exhaustion, DNS, clock), **tools** (AWS FIS, Chaos Mesh, LitmusChaos, Gremlin, toxiproxy), **game days** and **DR drills**, chaos in CI, and building the culture safely.
>
> **You need:** topics 23–24, 28–30, 41–42.
>
> **Part 8 · Security, DR and cost** · Previous: [43 · Cost and FinOps](43-cost-and-finops.md) · Next: [45 · Case: URL shortener](45-case-url-shortener.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Chaos engineering** = running controlled experiments that inject failures into a system to find weaknesses **before** they cause real outages. You test your resilience assumptions ("we survive losing an AZ") instead of trusting them.

```
1. Steady state:  define normal with metrics (checkout success ≥ 99.9%, p99 < 300 ms)
2. Hypothesis:    "If one AZ's pods die, steady state holds (traffic shifts, autoscaling covers)"
3. Inject:        kill all pods in AZ-a (small blast radius first; abort if SLI drops below threshold)
4. Observe:       did steady state hold? alerts fire? runbook correct?
5. Learn & fix:   found: HPA too slow, one service single-replica → fix → re-run
```

**Picture it.** A fire drill. You don't wait for a real fire to find out the emergency exit is blocked. You run a drill, announced or not, with someone ready to stop it, then fix what you learned.

**The one thing to know.** **Start small and controlled**: staging first, then production with a tiny blast radius, clear abort conditions, during working hours, with the owning team present. The goal is learning, not breaking things.

**Try it now (K8s staging):** delete one pod of a service while watching its error rate: `kubectl delete pod <one-pod> && watch kubectl get pods`. Did any request fail?

**You've got it if you can:** write a chaos experiment with hypothesis and abort conditions, and plan a game day.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **Chaos experiment** | A controlled fault injection with a hypothesis and measurement |
| **Steady state** | Measurable normal behaviour (usually SLIs) |
| **Blast radius** | What an experiment can affect (one pod, one AZ, 1% of users) |
| **Abort / stop condition** | Automatic rollback trigger (e.g. error rate > 1%, or an alarm in alarm state) |
| **Game day** | A planned team exercise simulating an incident/failure (technical + people + process) |
| **DR drill** | Exercise of the disaster recovery plan (topic 41) |
| **Fault injection** | The actual action: kill, delay, error, exhaust, partition |

Netflix popularized it (Chaos Monkey randomly terminating instances, ~2011). Today it's standard practice for critical systems.

---

## Why it exists

Redundancy and failover paths are **rarely exercised**, so they rot: misconfigured health checks, missing permissions in the DR region, timeouts set to defaults, fallbacks that throw exceptions, runbooks out of date, alerts that don't fire. You discover these during real incidents (expensive) or during experiments (cheap). Chaos experiments also train people (MTTR).

---

## Analogy

Fire drill (Core). Also **vaccination**: a small, controlled dose of the problem builds immunity to the real thing.

---

## How it works inside

### Fault catalogue

| Category | Faults | Tests |
|---|---|---|
| **Compute** | Kill instance/pod/container, node drain, Spot interruption | Redundancy, PDBs, graceful shutdown, rescheduling |
| **Zone/region** | Block all traffic in one AZ (subnet NACL/SG chaos), disable a region in routing | Multi-AZ design, N+1 capacity, failover runbooks |
| **Network** | Latency (+300 ms), packet loss, partition between services, DNS failure | Timeouts, retries, circuit breakers (topic 24) |
| **Dependency** | Return errors/timeouts from a dependency (DB, cache, third-party via proxy like toxiproxy) | Fallbacks, degradation, alerting |
| **Resource** | CPU hog, memory pressure, disk fill, file descriptor exhaustion | Limits, eviction behaviour, alerts |
| **State/data** | DB failover, cache flush, replica lag | Reconnect logic, stampede protection (topic 14) |
| **Time** | Clock skew | TLS, tokens, ordering (topic 26) |
| **Human/process** | On-call unreachable, runbook missing, credentials expired | Escalation, documentation |
| **Security** | Revoke a credential, simulate leaked key response | IR process |

### Experiment template

```
Name:            Redis primary failover during peak-like load
Owner:           payments team ; IC: <name> ; observers: SRE
Environment:     staging (round 1), production 10% traffic cell (round 2)
Steady state:    checkout success ≥ 99.9% (5-min), p99 < 400 ms, DB CPU < 70%
Hypothesis:      Redis failover completes < 30 s; checkout success stays ≥ 99.5%; DB CPU < 85%
Method:          AWS FIS action: trigger ElastiCache failover (or `redis-cli DEBUG SLEEP` in staging)
Blast radius:    one cache cluster (one cell)
Abort conditions:checkout success < 99% for 1 min OR DB CPU > 90% → stop, roll back (FIS stop condition = CloudWatch alarm)
Comms:           announced in #ops 1 day before; status: game day in progress
Results:         [observed metrics, timeline, alerts fired?]
Findings:        [e.g. app reconnect took 90 s due to DNS caching → action item]
```

### Maturity path

```
1. Tabletop exercises (talk through a scenario, no injection)
2. Game days in staging (planned, team present)
3. Production experiments with tiny blast radius, business hours, auto-abort
4. Regular automated experiments (e.g. weekly pod/instance kills in prod)
5. Chaos in CI/CD (resilience tests run on every release for critical paths)
6. Full DR/region failover drills in production
```

### Tools

| Tool | Scope |
|---|---|
| **AWS FIS** (Fault Injection Service) | EC2/ECS/EKS/RDS/network/AZ faults with CloudWatch-alarm stop conditions; scenario library (e.g. AZ power interruption) |
| **Azure Chaos Studio**, GCP (via Chaos Mesh/Litmus/Gremlin) | Cloud equivalents |
| **Chaos Mesh**, **LitmusChaos** | Kubernetes-native (pod kill, network chaos, stress, time, IO) as CRDs |
| **Gremlin** | Commercial, multi-platform, safety features |
| **toxiproxy**, Envoy fault injection, Istio `fault` | Latency/errors between services |
| **Chaos Monkey / Spinnaker** | Random instance termination |

### Game days

```
Before:  pick scenario (from risks / recent incidents / DR plan), objectives, roles (IC, operators, observers),
         success criteria, safety plan, notify stakeholders
During:  inject (or simulate), team responds as in a real incident, scribe records timeline,
         facilitator may add twists ("Slack is down", "the on-call lead is unavailable")
After:   blameless review, measured RTO/RPO vs target, action items with owners (topic 30)
```
Good scenarios: "AZ outage at peak", "primary DB lost", "region failover", "certificate expired", "CI is down and we need a hotfix", "credentials leaked", "a dependency returns 30 s latency".

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Where | Staging (safe, less realistic) | Production (real, needs safeguards) → start staging, graduate to prod |
| Announced vs surprise | Announced (team learns calmly) | Unannounced (tests detection and real response; only with mature teams and leadership buy-in) |
| Manual vs automated | Manual game days (deep learning) | Continuous automated experiments (regression protection) ✅ both |
| Scope | Single component | Whole scenario (AZ, region) |

---

## Failure modes (of chaos practice)

| Failure | Consequence | Prevent |
|---|---|---|
| **No abort conditions** | Experiment becomes a real outage | Automatic stop conditions on SLIs |
| **Running during peak or a freeze** | Customer impact | Schedule in low-risk windows; respect error budget (don't experiment when budget is exhausted) |
| **No steady-state metrics** | Can't tell if anything broke | SLOs first (topic 28) |
| **Findings not fixed** | Same weakness in the next real incident | Action items tracked like postmortems |
| **Chaos theater** | Only killing pods that obviously restart | Target real risks: dependencies, AZs, DB failover, regional DR |
| **Surprising other teams** | Confusion, parallel incident response | Announce, coordinate, status channel |

---

## Where you'll use it

- **Validating designs** in every case study ("how do you know it survives an AZ loss?" → "we test it monthly with FIS").
- **DR tests** (topic 41), **load + failure tests** (topic 31).
- **K8s:** PDBs, probes, topology spread verified by killing nodes/pods.
- **Ansible/Jenkins/GHA:** scheduled chaos jobs in pipelines; Ansible `serial` rolling restarts are mini-experiments.

---

## Design exercise

**Prompt:** Your team claims its EKS-based API "survives an AZ failure". Design a program to prove it, starting next month.

<details>
<summary><b>Model answer</b></summary>

- **Week 1 – prerequisites:** SLOs and dashboards for the API (success, latency); per-AZ metrics; verify topology spread constraints, PDBs, N+1 capacity per AZ on paper; runbook "AZ impairment".
- **Week 2 – tabletop:** walk through an AZ failure: what happens to pods, nodes (Karpenter launches in other AZs?), RDS (Multi-AZ failover if the primary is in that AZ), ElastiCache, NAT (per AZ?), ALB (cross-zone), EBS-backed stateful pods (stuck in that AZ!). Record gaps.
- **Week 3 – staging experiment:** AWS FIS "AZ availability: power interruption" scenario (or network disruption to one AZ's subnets + node termination) under a k6 load at expected peak. Stop condition: success < 99% for 2 min. Measure: error spike duration, time to full capacity, RDS failover time, alerts fired.
- **Week 4 – fix findings:** e.g. HPA min replicas too low to absorb, a single-AZ Redis, StatefulSet pods unable to move, NAT single AZ in staging.
- **Month 2 – production experiment:** business hours, announced, start with a **partial** AZ fault (pods in one AZ for one service), then expand; auto-abort via CloudWatch alarm on the SLI.
- **Ongoing:** quarterly AZ game day; weekly automated pod/node termination in prod; results recorded (measured impact vs SLO). The claim becomes evidence: "last AZ test: 0.3% errors for 70 s, within budget."
</details>

---

## Quick recap

- Chaos engineering = controlled experiments: steady state → hypothesis → inject → observe → fix.
- Control blast radius, use automatic abort conditions, start in staging, graduate to production.
- Inject realistic faults: AZ loss, dependency latency/errors, DB failover, resource exhaustion, DNS, clocks.
- Game days exercise people and runbooks too; DR drills measure real RTO/RPO.
- Findings become tracked action items; continuous experiments prevent regressions.

### Interview questions

1. **Q: Isn't breaking production on purpose risky?**
   A: Less risky than discovering weaknesses during a real outage. Experiments are controlled: hypothesis, small blast radius, business hours, people watching, automatic stop conditions on SLIs, and only when the error budget allows. You start in staging and expand gradually.

2. **Q: What would your first chaos experiments be?**
   A: The ones matching our biggest assumed protections: killing pods/nodes of critical services (redundancy, PDBs, graceful shutdown), a database failover (reconnect behaviour), injected latency on a key dependency (timeouts and fallbacks), and an AZ impairment game day. Each with steady-state SLIs and abort conditions.

3. **Q: How do chaos engineering and SLOs relate?**
   A: SLIs define the steady state and abort conditions; the error budget decides when experiments are allowed (not when budget is exhausted) and how much risk is acceptable. Experiment results feed reliability work prioritized by the same error budget policy.
