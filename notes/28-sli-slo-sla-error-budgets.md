# 28 · SLI, SLO, SLA and Error Budgets

> **Level:** ★★★ advanced
>
> **You will learn:** the three terms precisely (**SLI** = Service Level Indicator, **SLO** = Service Level Objective, **SLA** = Service Level Agreement), how to **choose good SLIs** (user-centric, measured where users feel it), how to **set an SLO** (not 100%!), the **error budget** and its **policy**, **burn rate** math, **SLO windows** (rolling vs calendar), SLOs for **non-request** systems (pipelines, batch, data freshness), **dependency SLOs**, and how SLOs drive decisions about releases and reliability work.
>
> **You need:** topics 23, 27.
>
> **Part 5 · Observability and SRE** · Previous: [27 · Observability](27-observability.md) · Next: [29 · Alerting and on-call](29-alerting-and-on-call.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.**
```
SLI  = a measurement of good service.        "% of checkout requests that succeed in < 500 ms"
SLO  = the internal target for the SLI.      "99.9% over 30 days"
SLA  = a contract with consequences.         "99.5% per month, or customers get 10% credit"
Error budget = 100% − SLO.                   99.9% → 0.1% of requests may fail ≈ 43 min/month equivalent
```

**Picture it.** A bus company. SLI: % of buses arriving within 5 minutes of schedule. SLO (internal goal): 99%. SLA (promise to the city, with fines): 97%. Error budget: 1% of buses may be late. If they've already used it this month, no experiments with new routes until things are stable.

**The one thing to know.** **100% is the wrong target.** Users can't tell 99.99% from 100% (their own Wi-Fi fails more often), and chasing it freezes all change. The error budget turns reliability into a shared, numeric trade-off: **budget left → ship faster; budget spent → focus on reliability**.

**Try it now:** for one service, write an SLI as `good events / valid events`, measured at the load balancer.

**You've got it if you can:** write SLIs for an API and a pipeline, compute an error budget in minutes and requests, and explain burn rate.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Full form | Owned by | Example |
|---|---|---|---|
| **SLI** | Service Level Indicator | Engineering | Ratio of successful requests faster than 300 ms |
| **SLO** | Service Level Objective | Engineering + product | 99.9% of requests good over a rolling 28 days |
| **SLA** | Service Level Agreement | Business/legal | 99.5% monthly; below → service credits |
| **Error budget** | 1 − SLO | Shared | 0.1% of requests in the window |
| **Burn rate** | How fast the budget is being consumed relative to "evenly over the window" | — | Burn rate 1 = budget lasts exactly the window; 10 = gone in 1/10 of it |

**Ordering rule:** SLA < SLO < what you actually achieve. The SLO is stricter than the SLA so you get warning before you owe customers money.

**SLI form (recommended):** `SLI = good events / valid events × 100%` (request-based). Valid events exclude things that aren't your fault or aren't real usage (e.g. 4xx from bad client input, health checks, synthetic traffic sometimes).

---

## Why it exists

Without SLOs:
- Reliability is argued by feelings ("the site feels slow").
- Ops wants zero change; dev wants to ship. Endless conflict.
- Alerts fire on CPU and every blip; on-call burns out (topic 29).
- Nobody knows whether to spend next sprint on features or reliability.

SLOs give a **shared number** that decides these. Google's SRE practice popularized this.

---

## Analogy

Bus company (Core). The **error budget** is like a monthly mobile data plan: you can spend it on anything (risky releases, migrations, experiments, unexpected incidents). If you burn it in the first week, you're careful for the rest of the month.

---

## How it works inside

### Choosing SLIs

Measure **what users experience**, as **close to the user** as practical:

| Service type | SLI kinds | Example SLI |
|---|---|---|
| Request/response (API, web) | **Availability**, **latency** | Good = status not 5xx AND latency < 300 ms, measured at the LB/gateway |
| Data pipeline | **Freshness**, **correctness**, **coverage** | % of minutes where the dashboard data is < 10 min old |
| Batch job | **Completion within deadline**, success | % of nightly runs completed by 06:00 |
| Storage | **Durability**, availability of reads/writes | % of objects readable; no data loss |
| CI/CD platform (internal) | **Queue time**, success not due to platform, duration | % of jobs starting within 2 min; % of job failures not caused by infra |
| Streaming/consumer | **Lag/end-to-end latency** | % of events processed within 60 s |

Where to measure (from best to worst for user truth): **client/RUM** (real but noisy) → **edge/LB/gateway** ✅ (great default) → **service metrics** (misses failures before the service) → **synthetic probes** (good for low-traffic services).

**Critical user journeys:** pick SLOs for the few flows that matter (login, search, add to cart, checkout), not every endpoint.

### Setting the SLO

1. Look at history: what have we achieved over the last 3 months? Start near that (slightly below if users are happy).
2. Ask: what do users need? What do dependencies allow (topic 23)?
3. Choose a window: **rolling 28 or 30 days** (smooth, always current) or calendar month (matches SLAs/reporting).
4. Review quarterly. Tighten if users complain while you're within SLO; relax if you're spending too much effort with no user benefit.

### Error budget math

```
SLO 99.9%, rolling 30 days, 50 M requests/month
Budget = 0.1% × 50 M = 50,000 failed requests
Time equivalent (if everything failed): 0.001 × 30 × 24 × 60 = 43.2 minutes

Incident: 20 minutes at 30% errors on 1,200 req/s
   failed = 20 × 60 × 1,200 × 0.30 ≈ 432,000 → 8.6× the monthly budget  (SLO missed)
Small blip: 2 minutes at 5% errors → 2×60×1,200×0.05 = 7,200 → 14% of budget
```

### Burn rate

```
burn rate = (observed error rate) / (1 − SLO)
SLO 99.9% → allowed error rate 0.1%
Observed 1% errors → burn rate 10 → a 30-day budget lasts 3 days
Observed 14.4% errors → burn rate 144 → budget gone in 5 hours
```
Alerts on burn rate (topic 29) catch both fast disasters and slow leaks, with far fewer false alarms than threshold alerts.

### Error budget policy

A written agreement (engineering + product + leadership), for example:

| Budget state | Action |
|---|---|
| > 50% left | Normal feature work; experiments OK |
| 25–50% left | Extra review for risky changes |
| < 25% left | Only low-risk releases; prioritize reliability fixes from recent incidents |
| Exhausted | **Release freeze** except fixes and security patches until budget recovers; postmortem action items get top priority |
| Single incident > 20% of budget | Mandatory postmortem (topic 30) |

The policy only works if leadership agrees in advance. Otherwise it's ignored at the first deadline.

### SLO implementation (Prometheus sketch)

```promql
# SLI: availability over 30 days (good = non-5xx) at the gateway
sum(rate(gateway_requests_total{service="checkout",code!~"5.."}[30d]))
/
sum(rate(gateway_requests_total{service="checkout"}[30d]))
```
Tools: **Sloth**, **Pyrra** (generate recording rules + alerts from SLO specs), **OpenSLO** spec, Google Cloud SLO monitoring, Datadog/Grafana SLOs, CloudWatch Application Signals.

### Dependencies and internal platforms

- Your SLO can't be tighter than your hard dependencies allow (topic 23 math), unless you design around them.
- **Platform teams** (CI, K8s, gateways) publish SLOs to their internal customers: "99.9% of builds start within 2 minutes", "K8s API 99.95% available". This makes platform reliability negotiable and measurable (topic 39).

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Window | Rolling 28/30 d: smooth, always current | Calendar month: matches SLA/billing, resets abruptly |
| SLI type | Request-based: accurate for traffic | Time-based (good minutes): better for low-traffic services where one failed request is 10% |
| Measurement point | Edge/LB: close to user | Service: easier, misses upstream failures |
| Latency SLI | Threshold ("< 300 ms") as part of "good" ✅ | Percentile target ("p99 < 300 ms"): harder to turn into a budget |
| Number of SLOs | A few per critical journey ✅ | Every endpoint: noise, nobody looks |

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **SLO = 100% or 99.999% "because important"** | Constant breaches, ignored SLOs, change paralysis | Set from history and user need |
| **SLI measures servers, not users** (CPU, uptime of pods) | Green SLO while users fail | Request-based SLIs at the edge, synthetic for journeys |
| **No error budget policy** | SLO is a pretty graph, nothing changes | Written policy signed by leadership |
| **Too many SLOs** | Nobody reads them | Critical user journeys only |
| **SLA ≥ SLO** | Breach the contract before internal warning | SLA looser than SLO |
| **Excluding too much from "valid"** | SLO looks great, users unhappy | Review exclusions; only exclude genuine client errors |
| **Low traffic noise** | 1 failure = SLO breach at night | Time-based SLIs, synthetic traffic, longer windows |

**3 AM relevance:** SLOs decide **whether** to wake someone: page only when the error budget is burning fast (topic 29). A pod restart that users never notice shouldn't page anyone.

---

## Where you'll use it

- **Every case study** has SLOs in the observability section.
- **Release decisions** in CI/CD: gate production deploys on remaining budget (topic 34).
- **Platform teams:** publishing SLOs for CI, clusters, gateways (topics 39, 47).
- **Incident severity** tied to budget impact (topic 30).
- **Interviews:** "How do you define reliability for this system?" → SLIs + SLOs + budget policy.

---

## Design exercise

**Prompt:** Define SLIs and SLOs for (a) a public REST API for orders, (b) the company's CI platform, (c) a nightly data export to a partner due at 07:00. Include the error budget per month for (a) at 30 M requests/month.

<details>
<summary><b>Model answer</b></summary>

**(a) Orders API**
- SLI availability: `non-5xx responses / all requests excluding 4xx`, at the gateway.
- SLI latency: `requests completed < 400 ms / all valid requests`, for `POST /orders` and `GET /orders/{id}`.
- SLOs: availability 99.9%, latency 99% (rolling 30 days).
- Budget: 0.1% × 30 M = **30,000** failed requests/month (~43 min of full outage equivalent); latency budget 1% = 300,000 slow requests.
- Policy: freeze risky releases when < 25% left.

**(b) CI platform (internal)**
- SLI queue time: `% of jobs that start within 2 minutes of being queued`. SLO 99%.
- SLI platform reliability: `% of jobs that don't fail due to platform causes` (runner lost, cache/registry errors, out of disk; classified by error type). SLO 99.5%.
- SLI duration (informational, maybe SLO later): p90 of standard pipeline duration.

**(c) Nightly export**
- SLI: `exports delivered and validated (row counts, checksum) by 07:00 / scheduled exports`. SLO: 99% of days per quarter (≈ at most 1 late day per quarter).
- Plus a freshness alert when the job hasn't completed by 05:30 (time to fix before the deadline).

Trade-off notes: latency threshold is per-endpoint because a list endpoint is naturally slower; SLO windows rolling for the API, quarterly for the daily job (a monthly window with 30 events is too noisy).
</details>

---

## Quick recap

- SLI = good/valid events; SLO = internal target; SLA = contract (looser than SLO); error budget = 1 − SLO.
- Choose SLIs at the user's edge for critical journeys; set SLOs from history and need, never 100%.
- Burn rate = error rate ÷ budget rate; alert on burn rate.
- An error budget policy (agreed with leadership) turns the number into decisions: ship vs stabilize.
- Pipelines, batch jobs and platforms get SLOs too: freshness, deadlines, queue time.

### Interview questions

1. **Q: What's the difference between SLO and SLA, and which is stricter?**
   A: An SLO is an internal reliability target; an SLA is an external contract with penalties. The SLO should be stricter so you get an internal warning and time to react before breaching what you promised customers.

2. **Q: Your SLO is 99.9% and you've used the whole error budget in week two. What happens?**
   A: Per the error budget policy: freeze non-essential releases, put postmortem action items from the incidents first, and focus engineering on reliability until the budget recovers. If the SLO is constantly blown by things that don't hurt users, we revisit the SLI or target instead.

3. **Q: How would you pick SLIs for a Kubernetes platform team?**
   A: From the perspective of the teams using it: API server availability and latency for deploys, pod scheduling latency (time from create to running), ingress/gateway request success rate, and CI deploy success rate attributable to the platform. Not node CPU; those are causes, not user experience.
