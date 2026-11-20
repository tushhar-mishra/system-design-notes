# 30 · Incident Management and Postmortems

> **Level:** ★★ intermediate
>
> **You will learn:** what an **incident** is and how to **classify severity**, the **incident lifecycle** (detect → triage → mitigate → resolve → learn), the roles (**Incident Commander, Ops/Tech lead, Communications lead, Scribe**), why you **mitigate before you find the root cause**, the standard **mitigation toolbox** (rollback, failover, scale, shed, feature flag), **communication** (internal and status page), **blameless postmortems** with **contributing factors**, writing **action items that get done**, and metrics (**MTTD, MTTA, MTTR**).
>
> **You need:** topics 23, 24, 28–29.
>
> **Part 5 · Observability and SRE** · Previous: [29 · Alerting](29-alerting-and-on-call.md) · Next: [31 · Capacity planning](31-capacity-planning-and-performance-testing.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** An **incident** is an unplanned event that hurts (or threatens) users or the business and needs a coordinated response. The goal during the incident is **restoring service**, not understanding everything.

```
Detect → Declare → Triage (severity) → MITIGATE (stop the bleeding) → Resolve → Postmortem → Action items
```

**Roles in a big incident:**
- **Incident Commander (IC):** coordinates, decides, keeps the big picture. Does **not** type commands.
- **Ops / Tech lead:** investigates and executes changes.
- **Comms lead:** updates stakeholders and the status page on a schedule.
- **Scribe:** records the timeline.

**Picture it.** A fire brigade. The commander stands back with the radio, assigns crews, and decides; the crews fight the fire; a liaison talks to the residents and the press. Nobody holds an investigation into the cause while the building burns.

**The one thing to know.** **Mitigate first:** if something changed recently, **roll it back** even before you're sure it's the cause. Rollback is usually safe and fast; debugging under pressure is slow.

**Try it now:** write your team's top 5 "big red buttons": the fastest safe mitigations (rollback command, failover procedure, feature flag kill switch, traffic shift, scale-up).

**You've got it if you can:** run an incident call with roles, define severities, and write a blameless postmortem with useful action items.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**Severity levels (example; adapt to your company):**

| Sev | Definition | Example | Response |
|---|---|---|---|
| **SEV1** | Critical user-facing outage or data loss/security breach | Checkout down for all users; data leak | Page IC + on-call immediately, exec notification, status page within 15 min, all hands as needed |
| **SEV2** | Major degradation or partial outage | 20% of users can't log in; one region down | Page on-call + IC, status page |
| **SEV3** | Minor impact, workaround exists | Slow reports, one non-critical feature broken | On-call handles in hours |
| **SEV4** | No user impact yet / near miss | Backup failed once, redundancy lost | Ticket |

Severity can change during the incident. **When in doubt, declare higher and downgrade later.** Declaring is cheap; missing a SEV1 is expensive.

**Metrics:**
- **MTTD** (Mean Time To Detect): problem start → detection.
- **MTTA** (Mean Time To Acknowledge): alert → human ack.
- **MTTM** (to Mitigate): detection → users no longer impacted.
- **MTTR** (to Resolve/Recover): often used for "to mitigate"; define it explicitly in your org.

---

## Why it exists

Without a process, big incidents turn into chaos: ten people debugging the same thing, nobody updating customers, conflicting changes made at once, decisions not made, and the same incident happening again next month. A light, practised process shortens outages and builds trust. Interviewers for senior roles almost always ask "tell me about an incident you handled".

---

## Analogy

Fire brigade (Core). The **postmortem** is like aviation accident investigation: the goal is to understand **how the system allowed the error** (procedures, tooling, design), not to punish the pilot, because punishment makes people hide mistakes and the next crash isn't prevented.

---

## How it works inside

### The lifecycle step by step

```
1. DETECT     alert / synthetic check / customer report / engineer notices
2. DECLARE    open incident channel (#inc-2026-11-20-checkout), page IC, set severity
3. TRIAGE     what's the user impact? since when? what changed? (deploys, config, traffic, dependencies)
4. MITIGATE   stop the bleeding (toolbox below); verify with the SLI
5. COMMUNICATE every 15–30 min internally; status page updates for customers
6. RESOLVE    service stable, monitoring confirms; downgrade/close
7. FOLLOW UP  postmortem within ~5 business days; action items tracked to done
```

### First 10 minutes (the IC's checklist)

1. Acknowledge, open the incident channel and a call. Announce: "I'm IC."
2. **Impact statement:** what users see, how many, since when.
3. Assign Ops lead, Comms lead, Scribe (in small teams one person may cover two).
4. **What changed?** Check deploy history, config/flag changes, infra changes (Terraform applies), traffic, dependency status pages.
5. Decide the first mitigation. Ask: "What's the fastest safe action to reduce impact?"
6. Set the update cadence ("next update in 15 minutes").

### The mitigation toolbox (fastest first)

| Action | When | Notes |
|---|---|---|
| **Roll back** the last deploy/config/flag | Something changed recently | Fastest win; needs one-click rollback (topic 34) |
| **Kill switch / feature flag off** | New feature misbehaving | Topic 57 |
| **Fail over** (DB, region, AZ, provider) | Infrastructure component failed | Practised runbook (topic 41) |
| **Shift traffic** away from a bad AZ/region/cell | Localized failure | DNS weights, LB, mesh |
| **Scale up/out** | Capacity exhaustion | May not help if bottleneck is downstream (topic 11) |
| **Shed load / rate limit** | Overload, retry storm, abusive client | Topic 24–25 |
| **Restart** | Memory leak, stuck process | Buys time; capture a heap dump first if possible |
| **Disable non-critical features** | Protect the core path | Degradation plan |
| **Block bad traffic at edge** | Attack, bot | WAF rule |

⚠️ **One change at a time**, announced in the channel, so you know what helped. ⚠️ **Don't fix forward under pressure** unless rollback is impossible (e.g. a DB migration already ran and old code can't read new data; that's why migrations must be backward compatible, topic 35).

### Communication

- **Internal updates** (template): *Status: investigating/identified/monitoring/resolved. Impact: … Current action: … Next update: HH:MM.*
- **Status page** (customers): plain language, no blame, no internal details, regular updates even if "still investigating".
- **Stakeholders:** support (so they can answer customers), leadership (for SEV1/2), affected internal teams.

### Blameless postmortem

**Blameless** = we assume people acted reasonably with the information they had; we look for **systemic** causes. "Human error" is never a root cause; ask why the system made the error easy and its impact large.

**Template:**
```
Title, date, severity, authors, status
Summary (3–4 lines): what happened, impact, duration, root/contributing causes, how resolved
Impact: users affected, duration, error budget consumed, revenue/SLA impact, data loss?
Timeline (UTC): detection, key decisions, mitigation, resolution (from the scribe's notes)
Contributing factors: technical + process + organizational (multiple, not one "root cause")
What went well / what went poorly / where we got lucky
Action items: each with owner, priority, due date, ticket link
   - Prevent (stop this class of failure)
   - Detect (find it faster)
   - Mitigate (reduce impact / recover faster)
```

**Techniques:** "5 whys" (keep asking why, but branch into several contributing factors, not one chain); timeline analysis; ask "what would have made this a non-event?"

**Good vs bad action items:**

| Bad | Good |
|---|---|
| "Be more careful with deploys" | "Add automated canary analysis with auto-rollback to payments-svc pipeline (owner: A, due: Dec 15)" |
| "Improve monitoring" | "Add burn-rate alert on checkout success SLI with runbook link" |
| "Fix the bug" | Fix the bug **and** add a test for this class; add a timeout to the provider client |

Track completion rate; review overdue items in a weekly ops review. Share postmortems widely so other teams learn.

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Roll back vs fix forward | Rollback: fast, known-good ✅ | Fix forward: only when rollback is impossible/riskier |
| Declare early vs wait | Early: more people, maybe false alarm (cheap) ✅ | Wait: slower response to real SEV1 |
| Dedicated IC rotation | Experienced coordination | Needs trained people |
| Everyone joins the call | More eyes | Noise; IC should limit active participants |
| Public postmortems | Customer trust | Legal/security review needed |

---

## Failure modes (of the process)

| Failure | Symptom | Fix |
|---|---|---|
| **Hero debugging, no IC** | Chaos, duplicate work, no comms | Always assign IC for SEV1/2 |
| **Root-causing before mitigating** | Long outages | "Mitigate first" norm, big-red-button runbooks |
| **Many simultaneous changes** | Can't tell what fixed/broke it | One change at a time, announced |
| **Silence to customers** | Trust loss, support overload | Comms lead, status-page cadence |
| **Blame culture** | Hidden mistakes, shallow postmortems | Blameless norms, leadership modelling |
| **Action items never done** | Repeat incidents | Owners, due dates, weekly review, error budget policy priority |
| **No practice** | Fumbling during real SEV1 | Game days, incident drills (topic 44) |
| **Rollback broken** | Mitigation impossible | Test rollback regularly; keep last N versions deployable |

---

## Where you'll use it

- **Every senior interview:** "Describe a major incident you led." Use: context → impact → your actions (mitigation first) → root/contributing causes → what you changed afterwards (measurable).
- **CI/CD:** one-click rollback and deploy freeze switches are incident tools (topics 33–34).
- **IaC:** "what changed?" includes Terraform applies → audit trail of applies (topic 36).
- **Kubernetes break-fix:** [K8s 84 · production scenarios](../../CKA-Certification-Course-2025/notes/84-production-break-fix-scenarios.md), [Docker 44a · runtime security IR](../../Docker/notes/44a-runtime-security-and-incident-response.md).

---

## Design exercise

**Prompt:** 14:05 alert: checkout errors at 25%. 13:58 payments-svc v2.14 deployed. 14:00 the payment provider's status page says "degraded performance". You're IC. Walk through the first 20 minutes, then outline the postmortem's action items assuming the cause was v2.14's new retry logic amplifying provider slowness.

<details>
<summary><b>Model answer</b></summary>

**First 20 minutes**
- 14:06 Ack, open `#inc-checkout-errors`, declare **SEV1** (checkout = revenue, 25% failing). Assign Ops lead (payments on-call), Comms lead (support manager), Scribe.
- 14:08 Impact: ~1 in 4 checkouts failing since ~14:00. Two candidate causes: our deploy at 13:58 and the provider degradation at 14:00.
- 14:09 Decision: **roll back v2.14** now (fast, low risk) while another engineer checks the provider dashboard and our provider-call latency.
- 14:10 Comms: internal update + status page "Some customers are unable to complete checkout. We're investigating. Next update 14:25."
- 14:14 Rollback complete; errors drop to 6% (provider still slow). Partial mitigation.
- 14:15 Further mitigation: enable the **secondary payment provider** for 50% of traffic via flag; reduce provider timeout fallback path.
- 14:20 Errors < 0.5%. Status: monitoring. Keep the incident open until the provider recovers; then downgrade.

**Postmortem action items (contributing factors: new retry logic retried 3× without backoff or budget, multiplying load on a degraded provider; no canary analysis caught it within 7 minutes; provider degradation not alerting us directly)**
- Prevent: retry policy library with exponential backoff + jitter + retry budget, mandatory for outbound calls (owner, date).
- Prevent: canary deploy with automated analysis on checkout SLI for payments-svc (topic 34).
- Detect: alert on provider latency/error rate per provider; subscribe to provider status webhooks.
- Mitigate: automatic failover to the secondary provider when the primary's error rate > X% (circuit breaker).
- Process: deploy freeze when a critical dependency reports degradation.
</details>

---

## Quick recap

- Incident = coordinated response to user/business impact; severity drives response; declare early.
- Roles: IC coordinates, Ops lead fixes, Comms lead informs, Scribe records.
- Mitigate before root cause: rollback, kill switch, failover, shift traffic, shed load; one change at a time.
- Regular internal and customer updates.
- Blameless postmortems with multiple contributing factors and specific, owned, tracked action items (prevent, detect, mitigate).

### Interview questions

1. **Q: You're paged for a major outage. What do you do first?**
   A: Acknowledge, establish impact (who, how bad, since when), declare severity and open a channel, assign roles if it's big, check what changed recently, and apply the fastest safe mitigation (often rollback) while communicating on a fixed cadence. Root cause comes after users are OK.

2. **Q: What makes a postmortem blameless, and why does it matter?**
   A: It focuses on how the system (tools, processes, design, incentives) allowed a reasonable person's action to cause harm, rather than on who made the mistake. Without blamelessness people hide information, the analysis stays shallow, and the same failure repeats. With it, you get honest timelines and systemic fixes like guardrails and automation.

3. **Q: How do you make sure postmortem action items actually get done?**
   A: Make them specific with owners and dates, link tickets, prioritize them via the error budget policy, review open items weekly in an ops review, and report completion rates to leadership. Prefer fewer, high-leverage items (automation, guardrails) over long lists.
