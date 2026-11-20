# 01 · How to Approach a System Design Interview

> **Level:** ★ foundation
>
> **You will learn:** what a system design interview is really testing, a **step-by-step plan for 45–60 minutes**, how to ask **clarifying questions**, the difference between **functional** and **non-functional requirements**, how to drive the conversation like a senior, how the DevOps/SRE flavour of the interview differs from the "build Twitter" flavour, and the **common mistakes** that sink good engineers.
>
> **You need:** nothing. This is the starting point. A few words used here (load balancer, cache, SLO) are explained in one line where they appear and fully later.
>
> **Part 0 · How to think** · Next: [02 · The map](02-request-journey-map.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A system design interview is not a quiz with one right answer. It is a **simulated design meeting**. The interviewer watches *how you think*: do you find out what's really needed, put numbers on it, choose a simple design, explain trade-offs, and know how it fails?

**The plan (45 minutes):**

```
 0–5   Clarify     → what are we building, for whom, how big, how reliable?
 5–10  Estimate    → requests/sec, storage, bandwidth (rough numbers)
10–20  High level  → boxes and arrows, walk one request through it
20–35  Deep dive   → 2–3 hardest parts: data, scaling, failure, deployment
35–42  Operate it  → monitoring, SLOs, DR, security, cost, 3 AM failures
42–45  Wrap up     → trade-offs you made, what you'd do next
```

**Picture it.** An architect meeting a family who wants a house. A bad architect starts drawing a 5-bedroom villa. A good one asks: how many people, what budget, which city, do you work from home? Then sketches a simple plan, explains why the kitchen is where it is, and what happens in an earthquake.

**The one thing to know.** **Requirements first, design second.** Every design choice must trace back to a requirement ("I'm adding a read replica *because* reads are 50× writes and we need p99 under 200 ms").

**Try it now:** take "design a URL shortener". Write five clarifying questions before reading further. Compare with the list below.

**You've got it if you can:** list the 6 phases with their minutes, give 3 functional and 3 non-functional requirements for any system, and name 5 common mistakes.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

A **system design interview** asks you to design a large software system (or a piece of infrastructure) in 45–60 minutes, on a whiteboard or a shared drawing tool. There is no code. You talk, draw boxes and arrows, and write numbers.

There are two flavours:

| Flavour | Typical prompt | What they expect |
|---|---|---|
| **Product system design** (software engineers) | "Design Twitter / a chat app / a URL shortener" | Data model, APIs, scaling reads and writes |
| **Infrastructure / DevOps / SRE design** (your interviews) | "Design a CI/CD platform for 500 engineers", "Design logging for 2,000 servers", "Make this app survive a region failure", "Deploy with zero downtime" | Everything above **plus**: how it's deployed, monitored, secured, recovered, paid for, and how it fails at 3 AM |

For senior DevOps roles you will get **both**. Many interviews start with a product-style system ("a highly available web app") and then push you into operations ("now the database region is gone, walk me through it").

> **Senior (5–6 years)** in this context means the interviewer expects you to **lead**: you set the scope, you choose, you justify with trade-offs and numbers, and you talk about operations without being asked. A mid-level candidate answers questions. A senior candidate runs the meeting.

---

## Why it exists

Real design work looks exactly like this: vague requirements, a time limit, many valid answers, and a hidden list of ways it can fail. Companies use this interview because it shows:

1. **Scoping.** Can you turn "build X" into a clear problem?
2. **Breadth.** Do you know the building blocks (load balancers, caches, queues, databases)?
3. **Depth.** Can you go deep on at least two areas without hand-waving?
4. **Judgement.** Do you choose the *simplest* design that meets the need, and know its trade-offs?
5. **Operations.** Do you think about monitoring, failure, cost and security? (This is where DevOps candidates win or lose.)
6. **Communication.** Can another engineer follow your reasoning?

---

## Analogy

**A doctor's appointment.**

- A bad doctor hears "my head hurts" and prescribes surgery.
- A good doctor asks questions first (where, how long, how bad, any other symptoms?), does a quick measurement (temperature, blood pressure), forms a hypothesis, explains the options ("rest is cheap and usually works; a scan costs more but rules out X"), and plans a follow-up.

Clarify → measure → simple plan → options with trade-offs → follow-up. That is the interview.

---

## How it works inside: the six phases

### Phase 1 · Clarify (0–5 min)

Ask questions until you can write two short lists.

**Functional requirements** = **what** the system does. Features. Verbs.
- "Users can shorten a URL." "Engineers push code and get a build within 5 minutes." "We store logs from all servers and can search them."

**Non-functional requirements** = **how well** it does it. Qualities. Numbers.
- **Scale:** how many users, requests per second, data per day?
- **Latency:** how fast must a response be? (Usually said as "p99 under 200 ms". **p99** = the 99th percentile: 99% of requests are faster than this. Full story in topic 27.)
- **Availability:** how much downtime is OK? ("99.9%" = about 43 minutes a month. Topic 23.)
- **Durability:** can we ever lose data? (Logs: maybe a little. Payments: never.)
- **Consistency:** must every reader see the newest data instantly? (Topic 19.)
- **Security / compliance:** PII (Personally Identifiable Information), PCI-DSS (card payments), GDPR (EU privacy law), data residency?
- **Cost:** is there a budget? Is this a startup or a bank?
- **Team / existing tech:** what cloud, Kubernetes already, how many engineers will run it?

**Clarifying questions that work for almost any prompt:**

| Question | Why you ask it |
|---|---|
| Who are the users and how many? | Sets scale. 500 internal engineers ≠ 50 M public users. |
| What are the 2–3 most important features? | Lets you cut scope to what matters. |
| What's the read:write ratio? | Read-heavy → caching and replicas. Write-heavy → queues, sharding. |
| What availability and latency do we need? | Decides redundancy (multi-AZ? multi-region?). |
| Can we lose data? How much? | Decides replication, backups, RPO (topic 41). |
| Global or one region? | Decides CDN, multi-region, data residency. |
| What already exists? (Cloud, K8s, CI tool) | Senior engineers build on what's there; they don't rewrite. |
| Is there a cost constraint? | Changes managed-vs-self-hosted and redundancy choices. |

> ⚠️ **Don't ask 20 questions.** Ask the 5–8 that change the design, then **state your assumptions** out loud and move on: "I'll assume 10 M daily users, read-heavy, 99.9% availability, one region to start. Tell me if you'd like something different."

### Phase 2 · Estimate (5–10 min)

Turn the numbers into **requests per second, storage, bandwidth**. You'll learn the method in topic 03. The point is not precision. The point is to find out **which part is hard**. "5,000 reads/second" fits in one database with a cache. "500,000 writes/second" does not, so you know sharding or a log-based design is coming.

### Phase 3 · High-level design (10–20 min)

Draw the **simplest thing that works** for the requirements. Usually 5–8 boxes. Then **walk one request through it**, out loud, numbered:

```
 [Users] ──1──▶ [DNS] 
    │
    └──2──▶ [CDN] ──3──▶ [Load balancer] ──4──▶ [App servers ×N] ──5──▶ [Cache]
                                                       │                      
                                                       └──6──▶ [Database (primary + replica)]
                                                       └──7──▶ [Queue] ──▶ [Workers]
```

1. The browser asks DNS (the internet's phone book, topic 07) for the IP address.
2. Static files come from the CDN (servers close to the user that cache content, topic 15).
3. API calls go to the load balancer (spreads requests over servers, topic 09).
4. ...and so on.

Also write down the **API** (2–4 endpoints) and the **data model** (main tables or objects) if it's a product-style question.

### Phase 4 · Deep dive (20–35 min)

Pick the **2–3 hardest parts** (your estimation told you which) and go deep. Often the interviewer picks. Typical deep dives:

- Data: which database, how to scale it, replication, sharding (topics 16–19).
- Hot paths: caching and its failure modes (topic 14).
- Async work: queues, retries, idempotency (topics 21–22).
- For DevOps prompts: how runners scale, how deployments roll back, how logs are buffered, how state is locked.

For each choice, say the **trade-off** in one breath: *"I'll use async replication to the replica. It's faster and the primary isn't blocked, but a failover could lose the last second of writes. For this system that's acceptable because..."*

### Phase 5 · Operate it (35–42 min)

This is the **DevOps/SRE differentiator**. Cover, briefly and without being asked:

- **Observability:** what metrics, what SLO (Service Level Objective, the reliability target, topic 28), what alerts.
- **Failure:** what happens when each box dies? Walk through "AZ down", "database primary dies", "bad deploy".
- **Deployment:** how changes reach production safely (canary, rollback, topic 34).
- **Security:** who can access what, encryption, secrets.
- **DR:** backups, RPO/RTO (how much data and time we can lose, topic 41).
- **Cost:** the 1–2 biggest cost drivers and how to cut them.

### Phase 6 · Wrap up (42–45 min)

Summarize in 30 seconds: what you built, the key trade-offs, and **what you'd do next** with more time ("I'd add a second region once we have the revenue to justify 2× cost").

---

## Options and trade-offs: interview styles and how to adapt

| Situation | What to do |
|---|---|
| Interviewer gives a very vague prompt | Spend more time clarifying; propose scope yourself. |
| Interviewer gives very specific numbers | Skip most questions, go straight to estimation. |
| Interviewer keeps interrupting with "what if X fails?" | They want depth on reliability. Follow them; it's a signal. |
| Interviewer is silent | Keep talking, check in every few minutes: "Shall I go deeper on the data layer or the deployment?" |
| You don't know a technology they name | Say so, then reason from principles: "I haven't run Cassandra in production, but as a leaderless store it would..." |
| Running out of time | Jump to Phase 5 bullets. Missing operations is worse than missing a detail. |

**Breadth vs depth.** Junior answers are wide and shallow. Strong senior answers are **wide enough** (every box present) and **deep in 2–3 places** with numbers and failure modes.

---

## Failure modes: the common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| Jumping to boxes in minute 1 | You design the wrong system. | Always clarify; state assumptions. |
| Naming tools instead of reasoning ("Kafka, Kubernetes, Redis!") | Shows buzzwords, not understanding. | Say *why*: the requirement → the property → the tool. |
| Over-engineering (multi-region active-active for an internal tool) | Shows poor judgement and ignores cost. | Start simple, scale when the numbers demand it. |
| No numbers | Can't justify any decision. | Do the estimation (topic 03). |
| Single points of failure left in | Reliability is your job. | Walk each box: "what if this dies?" |
| Ignoring operations | The DevOps part of the job is missing. | Phase 5 every time. |
| Ignoring the database | It's usually the hardest part. | Always say how data is stored, scaled and backed up. |
| Never stating trade-offs | Every choice has a cost; not knowing it is a red flag. | "I chose X over Y because…, the cost is…" |
| Going silent while thinking | Interviewer can't score what they can't hear. | Think out loud. |
| Defending a mistake | Seniors update their view. | "Good point. That breaks under X; I'd change it to Y." |
| Spending 20 minutes on one detail | No time for the rest. | Timebox; offer to return to it. |

**The 3 AM mindset.** For each component, imagine being paged at 3 AM because it broke. How would you *know* (alert)? How would you *stop the bleeding* (failover, rollback, shed load)? How would you *prevent* it next time? Saying this out loud is the single most "senior" habit you can show.

---

## Where you'll use it

- **Interviews:** obviously. Every case study in Part 9 uses this exact flow.
- **Design reviews at work:** an RFC (Request For Comments, a written design proposal) follows the same shape: context, requirements, options, decision, risks.
- **Incidents:** "clarify → measure → simplest fix → trade-offs" is also how you lead an incident (topic 30).
- **Platform work:** before building a CI/CD platform or a cluster, you gather requirements from teams the same way.

---

## Design exercise

**Prompt:** "Design a system to collect logs from all our servers."

Write: (a) six clarifying questions, (b) three functional and four non-functional requirements (with your assumed numbers), (c) the phase you'd spend most time on and why.

<details>
<summary><b>Model answer</b></summary>

**(a) Clarifying questions**
1. How many servers/containers, and roughly how many log lines (or GB) per day each?
2. Who uses the logs and for what: debugging (search recent logs), security audit (keep for years), or both?
3. How long must we keep them? Any compliance rule (e.g. 1 year for audit)?
4. How fast must a log be searchable after it's written (seconds, minutes)?
5. Can we lose some logs during an outage, or must audit logs never be lost?
6. Is it Kubernetes, VMs, or both? What cloud? Any existing tool (ELK, Loki, Datadog)?

**(b) Requirements (assumed)**
- Functional: collect logs from all hosts and pods; search by service/time/text; alert on log patterns.
- Non-functional: 2,000 hosts × 5 GB/day ≈ 10 TB/day; searchable within 60 s; 99.9% availability for ingest; keep 14 days hot, 1 year archive; audit logs must not be lost; cost under a stated budget.

**(c) Most time:** the **ingestion pipeline and storage tiers**. 10 TB/day is the hard part: buffering (a queue so a slow index doesn't drop logs), indexing cost, and moving old logs to cheap object storage. That decides both reliability and cost. (Full design: topic 49.)
</details>

---

## Quick recap

- The interview is a design meeting: clarify, estimate, simple design, deep dive, operate, wrap up.
- Functional = what it does. Non-functional = how well (scale, latency, availability, durability, consistency, security, cost).
- Every choice traces back to a requirement and comes with a stated trade-off.
- For DevOps/SRE roles, always cover observability, failure, deployment, security, DR and cost.
- Think out loud, state assumptions, timebox, and update your design when challenged.

### Interview questions

1. **Q: You're asked to "design a highly available web app". What do you do in the first five minutes?**
   A: I ask what "highly available" means in numbers (99.9% vs 99.99%), how many users and requests, read:write ratio, whether it's one region or global, whether data loss is tolerable, and what cloud and tools already exist. Then I state assumptions out loud, such as "10 M daily users, 99.95%, single region multi-AZ to start", so the interviewer can correct me before I draw anything.

2. **Q: How do you decide how deep to go on each component?**
   A: Estimation tells me where the hard part is: the component whose load or data size doesn't fit a simple setup. I keep everything else at box level and go deep on the 2–3 risky parts, including their failure modes. If the interviewer steers somewhere else, I follow; that's where they want signal.

3. **Q: The interviewer points out a flaw in your design. What do you do?**
   A: Acknowledge it, explain *why* it's a flaw in terms of a requirement, and adjust: "You're right, with a single NAT gateway an AZ failure cuts egress for all subnets. I'll put one NAT per AZ; it costs more, but it removes the cross-AZ dependency." Defending a mistake is a bigger red flag than making it.
