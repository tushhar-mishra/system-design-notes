# 29 · Alerting and On-Call Design

> **Level:** ★★ intermediate
>
> **You will learn:** what makes a **good alert** (urgent, actionable, user-impacting), **symptom vs cause** alerting, **page vs ticket**, **multi-window multi-burn-rate** SLO alerts with exact numbers, **alert routing, grouping, inhibition and silences** (Alertmanager), **runbooks**, **meta-monitoring** (dead man's switch), designing **on-call rotations** that don't burn people out, and measuring **alert quality**.
>
> **You need:** topics 27–28.
>
> **Part 5 · Observability and SRE** · Previous: [28 · SLOs](28-sli-slo-sla-error-budgets.md) · Next: [30 · Incident management](30-incident-management.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** An alert should wake a human **only** if users are hurt (or about to be) **and** a human needs to act **now**. Everything else is a ticket, a dashboard, or nothing.

```
PAGE   (wake someone):  error budget burning fast (users affected now), data loss risk, security
TICKET (next workday):  slow budget burn, disk 80% in 5 days, cert expires in 14 days
LOG/DASHBOARD only:     pod restarted once, CPU 85% on one node, a single 500
```

**Burn-rate alerts (SLO 99.9%, 30-day window):**
```
Page:   burn rate > 14.4 over 1 h  AND over 5 m   → 2% of monthly budget gone in 1 h
Page:   burn rate > 6   over 6 h  AND over 30 m  → 5% in 6 h
Ticket: burn rate > 1   over 3 d  AND over 6 h   → slowly heading to a breach
```

**Picture it.** A smoke alarm vs a "battery low" chirp. The smoke alarm must wake you; it must never go off for toast. The battery chirp can wait until morning. An alarm system that screams for toast gets its battery removed.

**The one thing to know.** **Alert on symptoms (what users feel), investigate causes (why).** "Checkout error rate high" is a page. "CPU 90%" is not, because high CPU with happy users is fine, and users can suffer with low CPU.

**Try it now:** count last month's pages for your team. How many needed urgent human action? That's your **alert precision**.

**You've got it if you can:** turn an SLO into burn-rate alerts, decide page vs ticket for 10 examples, and design a sane rotation.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **Alert rule** | A condition on telemetry that, when true for a duration, fires |
| **Page** | Urgent notification that interrupts a human (phone, PagerDuty/Opsgenie/incident.io) |
| **Ticket** | Non-urgent work item |
| **Symptom-based alert** | Fires on user-visible impact (errors, latency, freshness) |
| **Cause-based alert** | Fires on a possible cause (CPU, disk, pod restarts) |
| **Routing** | Sending alerts to the right team/channel by labels (`team`, `severity`) |
| **Grouping** | Combining related alerts into one notification (100 pods failing = 1 page) |
| **Inhibition** | Suppressing dependent alerts when a root one fires (cluster down → don't page for every service) |
| **Silence** | Temporarily muting alerts (planned maintenance) |
| **Runbook / playbook** | Document: what this alert means, how to check, how to mitigate |
| **Dead man's switch / watchdog** | An alert that is *always* firing; if it stops arriving, the alerting pipeline is broken |

---

## Why it exists

Bad alerting causes the two worst on-call outcomes:
1. **Missed incidents** (no alert for what users feel) → long MTTD, angry customers find it first.
2. **Alert fatigue** (too many noisy alerts) → people ignore or silence alerts, then miss the real one, and burn out and quit.

Good alerting is the bridge between SLOs (topic 28) and incident response (topic 30).

---

## Analogy

Smoke alarm vs battery chirp (Core). Also a **hospital monitor**: it alarms on vital signs crossing dangerous lines (symptoms), not on every lab value; nurses who hear false alarms all day stop reacting ("alarm fatigue" is a real patient-safety problem in hospitals).

---

## How it works inside

### Alert pipeline

```
[Prometheus / CloudWatch rules] ─fire─▶ [Alertmanager / routing]
       evaluates every 30–60 s            group by (alertname, cluster, service)
                                          inhibit dependents, apply silences
                                          route: severity=page,team=payments → PagerDuty "payments"
                                                 severity=ticket → Jira / Slack channel
                                          ─▶ [Paging tool] → on-call phone → escalate if no ack in 5–15 min
```

### Multi-window multi-burn-rate alerts (from the Google SRE Workbook)

Why not "alert if error rate > 1% for 5 min"? It's either too sensitive (pages for short blips that don't threaten the SLO) or too slow (misses slow burns). Burn-rate alerts tie urgency to **budget consumption**.

For SLO **99.9%** over **30 days** (budget = 0.1% error rate):

| Severity | Budget consumed | Long window | Short window | Burn rate | Error rate threshold |
|---|---|---|---|---|---|
| Page | 2% in 1 h | 1 h | 5 m | 14.4 | 1.44% |
| Page | 5% in 6 h | 6 h | 30 m | 6 | 0.6% |
| Ticket | 10% in 3 d | 3 d | 6 h | 1 | 0.1% |

- **Long window** = is this significant? **Short window** = is it *still* happening? (So the alert stops soon after recovery.)
- Burn rate 14.4 for 1 hour = 14.4/720 h ≈ 2% of a 30-day budget.

```yaml
# Prometheus rule sketch (error ratio recording rules assumed)
- alert: CheckoutErrorBudgetFastBurn
  expr: |
    slo:checkout_errors:ratio_rate1h > (14.4 * 0.001)
    and
    slo:checkout_errors:ratio_rate5m > (14.4 * 0.001)
  labels: { severity: page, team: payments }
  annotations:
    summary: "Checkout burning error budget 14x (2% of monthly budget per hour)"
    runbook_url: https://runbooks.internal/checkout/error-budget
    dashboard: https://grafana.internal/d/checkout
```

### What else deserves a page (beyond SLO burn)

- **Imminent, unrecoverable harm:** disk will fill in < 4 h on a database; certificate expires in < 48 h and auto-renew failed; backups failing for > 24 h on a critical DB (not urgent at 3 AM? → ticket with escalation).
- **Security:** confirmed credential leak, unusual IAM activity on root.
- **Data loss/integrity risk:** replication broken for the only replica, CDC slot lag filling DB disk.
- **Dead man's switch** stopped (alerting broken).

Rule of thumb: **every page must have a runbook and require human judgement**. If the fix is always the same command, automate it instead.

### Good alert anatomy

```
Title:       [PAGE] checkout: error budget fast burn (14x) — prod / ap-south-1
What:        1.6% of checkout requests failing (SLO allows 0.1%) for 12 min
Impact:      ~1 in 60 customers can't pay
Dashboard:   link (pre-filtered)
Runbook:     link: check recent deploys → roll back; check payment provider status; ...
Recent changes: payments-svc v2.14 deployed 14:02 by CI
```

### On-call design

| Practice | Why |
|---|---|
| **Follow the sun** (hand-off across time zones) where possible | Nobody paged at 3 AM routinely |
| Rotations of **≥ 6–8 people** for 24/7 primary | Weekly shifts once every ~2 months; sustainable |
| **Primary + secondary** | Escalation if primary doesn't ack; backup for big incidents |
| **Target ≤ 2 incidents per 12-h shift** (Google SRE guidance) | Enough time to handle, follow up and write postmortems |
| Hand-off notes each shift | Context continuity |
| Compensation / time off after night pages | Retention and fairness |
| **"You build it, you run it"** | Teams that ship a service carry its pager → incentive to make it reliable |
| Review every page weekly | Kill or fix noisy alerts; track precision |

### Measuring alert quality

- **Precision:** % of pages that were real and actionable (aim > 80–90%).
- **Recall:** % of user-impacting incidents that were alerted (not found by customers).
- **Pages per shift**, **night pages**, **time to acknowledge**, **MTTD**.

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Alert type | Symptom/SLO burn (fewer, meaningful pages) ✅ for paging | Cause-based (earlier warning, noisy) → tickets/dashboards |
| Threshold alerts | Simple, intuitive | Don't scale with traffic; noisy |
| Burn-rate alerts | Tied to user impact; self-scaling | Need SLOs and recording rules; less intuitive |
| Low-traffic services | Burn rate noisy (1 error = 10%) | Use synthetic probes, time-based SLIs, minimum request counts in rules |
| Tooling | Alertmanager + PagerDuty/Opsgenie/incident.io | Cloud-native (CloudWatch alarms → SNS) for simpler setups |

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Alert fatigue** | Ignored pages, slow ack, burnout | Weekly review, delete non-actionable alerts, convert to tickets |
| **Missing alerts** | Customers report outages first | Symptom/SLO alerts on every critical journey; synthetic checks |
| **Alert storm** | 300 notifications during one outage | Grouping, inhibition (cluster-level alert inhibits service alerts) |
| **Alerting pipeline down** | Silence during an incident | Dead man's switch to an external service (e.g. healthchecks, PagerDuty heartbeat), HA Alertmanager |
| **Flapping alerts** | Fire/resolve repeatedly | `for:` durations, short+long windows, hysteresis |
| **No runbook** | Long MTTR, inconsistent responses | Runbook link required for every page-level rule (lint in CI) |
| **Wrong routing** | Platform team paged for app bugs | `team` labels from service catalog; ownership checked in CI |
| **Silences forgotten** | Real incident muted | Silences with expiry and owner; report active silences |

**3 AM:** you get paged. Ack → open the linked dashboard → check "what changed" → mitigate (rollback, failover, scale, shed) → declare an incident if it's big (topic 30). Next morning: was this page worth waking up for? If not, fix the alert.

---

## Where you'll use it

- **Prometheus/Alertmanager** on K8s (kube-prometheus-stack rules, PrometheusRule CRDs) ([K8s 80](../../CKA-Certification-Course-2025/notes/80-production-observability.md)).
- **CloudWatch alarms** via Terraform (`aws_cloudwatch_metric_alarm`, composite alarms).
- **CI/CD notifications** are not pages: build failures go to chat ([Jenkins 71 · notifications](../../Jenkins/notes/71-notifications.md)).
- **Alerting as code:** rules in Git, reviewed and unit-tested (`promtool test rules`) in CI.
- Every case study's observability section.

---

## Design exercise

**Prompt:** A team gets ~40 pages a week. Top sources: "pod restarted" (15), "CPU > 80%" (10), "5xx > 1% for 1 min" (8), "disk > 85%" (4), "certificate expires in 30 days" (3). They have an SLO of 99.9% on their API. Redesign their alerting.

<details>
<summary><b>Model answer</b></summary>

- **Pod restarted (15):** not a page. Kubernetes self-heals; a single restart doesn't hurt users behind replicas. → Dashboard + a **ticket** if a pod is in CrashLoopBackOff > 15 min or restarts > 10/h (`KubePodCrashLooping`). If restarts cause user errors, the SLO alert catches it.
- **CPU > 80% (10):** remove as a page. CPU is a cause. → Capacity dashboard; ticket if sustained > 85% for 1 day at peak (topic 31). HPA should handle it.
- **5xx > 1% for 1 min (8):** replace with **multi-window burn-rate alerts** (14.4× over 1 h/5 m and 6× over 6 h/30 m → page; 1× over 3 d/6 h → ticket). Brief blips stop paging; real incidents page faster and with impact info.
- **Disk > 85% (4):** replace with **predictive** "disk full in < 4 h" (`predict_linear`) → page only for stateful components (DBs); otherwise ticket at "full in < 3 days".
- **Certificate 30 days (3):** → ticket at 21 days; **page** only at < 7 days *and* renewal failed.
- **Add:** synthetic check on the top journey (page if failing from 2+ regions for 5 min), dead man's switch, runbook links required.
- **Expected:** ~2–5 pages/week, each meaningful. Track precision weekly.
</details>

---

## Quick recap

- Page only for urgent, actionable, user-impacting problems; tickets for slow problems; dashboards for causes.
- Symptom (SLO burn) alerts for paging; cause metrics for investigation.
- Multi-window multi-burn-rate: 14.4× (1 h/5 m) and 6× (6 h/30 m) page; 1× (3 d/6 h) ticket.
- Group, inhibit, route by team; every page has a runbook; watch the watcher (dead man's switch).
- Sustainable on-call: enough people, few pages, reviewed weekly, ownership with the builders.

### Interview questions

1. **Q: Why alert on symptoms rather than causes?**
   A: Symptoms map directly to user impact, so every page matters; there are few of them, and they catch unknown failure modes. Causes like CPU or restarts are often harmless (and noisy) or miss problems entirely. Causes are for dashboards and investigation once a symptom alert fires.

2. **Q: Explain multi-window burn-rate alerting.**
   A: Alert when the error budget is being consumed much faster than sustainable, using a long window to confirm significance and a short window to ensure it's still happening. For a 99.9% 30-day SLO: page at 14.4× burn over 1 h and 5 m (2% budget per hour) and 6× over 6 h and 30 m; ticket at 1× over 3 days. This pages quickly for big outages, ignores small blips, and resets soon after recovery.

3. **Q: How do you reduce on-call burnout?**
   A: Cut noise (review every page, delete or downgrade non-actionable ones, automate repetitive fixes), page on SLO burn only, group and inhibit related alerts, have runbooks, size rotations so each person is on-call rarely, use follow-the-sun if possible, compensate night work, and make reliability work from incidents a priority via the error budget policy.
