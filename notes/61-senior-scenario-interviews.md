# 61 · Senior Scenario Interviews

> **Level:** ★★★ advanced
>
> **What this is:** open-ended, senior-style interview rounds of the kind DevOps/SRE/Platform interviewers actually run: **design rounds** that keep adding constraints, **"what if" failure rounds**, **troubleshooting rounds**, **behavioural-technical** questions (incidents, disagreements, trade-offs), and **mock interview transcripts** with model answers and the follow-ups to expect. Practise them out loud with a timer.
>
> **You need:** topics 01–60.
>
> **Part 10 · Interview readiness** · Previous: [60 · Question bank](60-interview-question-bank.md) · Next: [62 · Cheat sheet](62-cheat-sheet.md) · [Back to INDEX](../INDEX.md)

---

## How seniors are evaluated

| Signal | Mid-level answer | Senior answer |
|---|---|---|
| Scoping | Answers the question as asked | Clarifies, sets assumptions, narrows scope with the interviewer |
| Numbers | Few or none | Quick estimates that drive choices |
| Choices | Names tools | Explains why, alternatives, trade-offs, cost |
| Failure | Mentions redundancy | Walks failure modes, blast radius, detection, mitigation, 3 AM |
| Operations | Rarely | SLOs, alerts, deploys, DR, security, cost, ownership, without prompting |
| Communication | Waits for questions | Drives, checks in, summarizes |
| Pragmatism | Max architecture | Simplest thing that meets requirements; evolution plan |

---

## Scenario 1: "Design CI/CD for a growing startup" (escalating constraints)

**Round 1:** 30 engineers, 15 services, AWS. *Model:* GitHub Actions with hosted runners, reusable workflow template, build once → ECR (immutable tags), deploy to EKS via Argo CD, rolling updates with good probes, OIDC to AWS. Simple, cheap, little to operate.

**Round 2: "Now 300 engineers and queue times are 20 minutes."** *Model:* measure queue vs run time; move to self-hosted ephemeral runners (ARC on EKS, Karpenter, Spot), warm buffers, caching (deps, BuildKit registry cache, package proxy), per-team concurrency limits; publish CI SLOs (topic 47).

**Round 3: "An auditor asks you to prove what's running in prod came from reviewed code."** *Model:* branch protection + reviews, signed images with SLSA provenance, admission policy verifying signatures, GitOps PR history with approvals, retained audit logs (topics 32–33, 40).

**Round 4: "A deploy caused a 30-minute outage. What changes?"** *Model:* canary with automated analysis and auto-rollback, error-budget gate, region waves, expand/contract migrations, feature flags; measure change failure rate and time to restore (topics 34–35, 48).

**Follow-ups to expect:** cost per build? CI as SPOF during incidents (break-glass)? Monorepo builds? Secrets in CI?

---

## Scenario 2: "What if…" failure round on your own design

Interviewers take your HA web app (topic 46) and fire failures. Answer each with **impact → detection → automatic behaviour → human action → prevention**.

| "What if…" | Strong answer sketch |
|---|---|
| …the Aurora writer dies? | ~30 s failover; RDS Proxy smooths; idempotent retries; alert if > 2 min; postmortem if data lost (none, storage quorum) |
| …Redis is flushed at peak? | DB load spike; stampede protection and concurrency limits keep DB alive; degrade nice-to-have features; warm cache; alert on hit ratio |
| …an AZ goes dark? | LB drops targets in that AZ; N+1 capacity serves; Karpenter replaces nodes; Aurora fails over if writer there; ~1 min elevated errors |
| …your certificate expires? | Shouldn't: ACM auto-renew + expiry probes; if it happens, upload/issue emergency cert; postmortem on monitoring gaps |
| …a junior runs `terraform destroy` on prod network? | Shouldn't be possible: no laptop applies, policy on plan blocks destroys, deletion protection; if partially happened, re-apply from known-good commit, restore |
| …the payment provider is slow (10 s)? | 3 s timeout, circuit breaker, secondary provider or "pay later" degradation, idempotency keys, alert on provider SLI |
| …GitHub is down and you need a hotfix? | Argo CD rollback to previous digest without CI; break-glass build path documented |
| …the region dies? | DR runbook: promote Global DB secondary, scale pilot-light cluster, flip DNS; RTO target 4 h, practised 1.5 h |
| …traffic is 10× tomorrow (TV ad)? | Pre-scale (pods, nodes, DB readers), raise quotas, warm caches/CDN, waiting room, shed non-critical features, freeze deploys |

---

## Scenario 3: Troubleshooting round (live)

**Interviewer:** "Latency p99 for our API went from 200 ms to 4 s twenty minutes ago. You have dashboards, logs, traces, kubectl. Go."

**Model answer (out loud):**
1. "Impact first: is it all endpoints, all regions, all customers? Is the error rate also up?" (Interviewer: all endpoints, one region, errors normal.)
2. "What changed in that region in the last hour: deploys, config, flags, infra, traffic?" (Interviewer: no deploys; traffic normal.)
3. "Follow the request: edge latency vs LB TargetResponseTime: if LB target time is high, it's inside. Open a slow trace." (Interviewer: traces show all services slow at their DB spans.)
4. "So the shared component: the regional DB. USE on the DB: CPU, IO, connections, locks, replication." (Interviewer: CPU 40%, IO latency 50 ms (normally 1 ms), connections normal.)
5. "Storage latency: check the storage volume metrics / provider health for that AZ/region; also a long-running vacuum or backup? Check RDS events and AWS Health." (Interviewer: AWS Health shows EBS degradation in one AZ, the writer is there.)
6. "Mitigate: trigger a failover to a reader in a healthy AZ (writes blip ~30 s) since latency impact is ongoing; communicate; watch latency recover. Prevention: alert on DB IO latency, document 'failover away from degraded AZ' in runbook, consider zonal shift automation."

**Why it's strong:** scoped, checked changes, followed the request, used USE, mitigated with a reasoned trade-off, closed with prevention.

---

## Scenario 4: "Design logging for 5,000 servers" with a twist

After a solid design (topic 49), the interviewer says: **"Your budget is cut by 60%."**
*Model:* reduce volume first (drop health checks/debug, sample INFO for high-volume services, aggregate access logs into metrics), shorten hot retention (14 → 7 days) with archive in S3 and on-demand query (Athena), move from full-text index to label-indexed store for app logs, keep security/audit full fidelity (non-negotiable), per-team showback and quotas. State what we lose: slower ad-hoc full-text search, less history in the hot tier.

---

## Scenario 5: Kubernetes platform for 40 teams: the "multi-tenancy" grilling

**Questions likely:** How do you prevent one team from consuming all nodes? (quotas, LimitRanges, priority classes, Karpenter limits per NodePool). How do you stop team A from reading team B's secrets? (namespaces + RBAC, no cluster-wide secret read, workload identity per service, policy preventing hostPath/privileged). Network isolation? (default-deny policies, mesh authz). Upgrades with 40 teams? (deprecated API scans in CI, staged fleet upgrades, PDB hygiene enforcement). Noisy neighbour on the API server? (API Priority and Fairness, controller reviews). Paying for it? (OpenCost showback by namespace labels).

---

## Behavioural-technical questions (use STAR: Situation, Task, Action, Result)

1. **"Tell me about the worst incident you handled."**
   Structure: context and impact (numbers), your role (IC or ops lead), mitigation first (what and why), root/contributing causes, what you changed afterwards (measurable: MTTR, alerts, automation), what you learned. Avoid blaming people.

2. **"Tell me about a design trade-off you made that you'd change today."**
   Show judgement: e.g. chose self-hosted Kafka for cost, underestimated operational load; today would choose managed or SQS for that use case; quantify the cost of the mistake and how you migrated.

3. **"How did you convince teams to adopt your platform/tool?"**
   Discovery of pain, quick win with one team, metrics (queue time ↓, lead time ↓), docs and support, made the golden path easiest, guardrails mandatory only where needed.

4. **"Describe a time you pushed back on a deadline for reliability reasons."**
   Use the error budget policy/data, offer options (ship behind flag to 1%, reduce scope), agree on risk explicitly with stakeholders.

5. **"How do you decide between building and buying?"**
   Core differentiation vs commodity, total cost (people + infra + licences), time to value, lock-in/exit cost, team capability; usually buy commodity, build glue and governance.

---

## Mock interview transcript (condensed, 45 minutes)

**Prompt:** "Design a system to deploy configuration changes to 10,000 servers safely."

- **Clarify (4 min):** What config (nginx/app config files, kernel params)? Servers: VMs in 3 regions, mixed RHEL/Ubuntu. Frequency: dozens of changes/day. Safety: no fleet-wide outage from a bad change; rollback < 5 min; audit. Existing: Ansible, Git, Jenkins.
- **Estimate (3 min):** 10,000 hosts; push via Ansible at forks 100 → a full run of a 30 s play ≈ 10,000/100 × 30 s ≈ 50 min sequential batches; too slow for urgent rollbacks → pull model or higher parallelism with batching.
- **High level (8 min):** Config in Git (roles + inventories per env/region) → CI lint/test (ansible-lint, Molecule) → artifact (versioned role/bundle) → rollout controller applies in **waves**: canary hosts (1%) → one AZ → one region → all, with health checks between waves (service checks, error rates from monitoring) and automatic halt; hosts pull desired version via ansible-pull (or agent) on signal, reporting status. Rollback = point the wave back to previous version.
- **Deep dives (15 min):** idempotency and `--check --diff` previews; `serial` + `max_fail_percentage` as circuit breakers ([Ansible 45](../../Ansible/notes/45-strategies-forks-serial.md), [Ansible 31](../../Ansible/notes/31-error-handling.md)); validation (`nginx -t` via `validate:`); handlers to reload, not restart; drift detection with scheduled check runs ([Ansible 55](../../Ansible/notes/55-ansible-pull-and-drift.md)); secrets via Vault lookups; inventory from cloud tags; AAP/AWX for RBAC and audit ([Ansible 56](../../Ansible/notes/56-awx-and-aap.md)).
- **Operate (8 min):** SLI: % of hosts converged on desired version; alerts on stalled waves; audit trail; DR: Git mirrored, controller HA; security: least-privilege automation user, signed collections; cost: minor.
- **Wrap (2 min):** trade-offs: pull model scales and is resilient but slower to converge than push; waves add time but bound blast radius; immutable images (bake + replace) would remove most config drift long term.

**Likely follow-ups:** "A change is half applied when the controller dies?" (idempotent re-run converges; wave state persisted). "How do you handle a host offline during rollout?" (pull on boot, drift report). "Why not just bake images?" (great for stateless fleets; config changes then become deploys; some long-lived hosts still need CM).

---

## Practice plan

1. One case study (topics 45–57) per day for two weeks, 45 minutes, out loud, recorded.
2. After each: self-review with topic 59's checklist and topic 01's mistake list.
3. Every third day: a troubleshooting scenario from topic 58 in 10 minutes.
4. Prepare 4 STAR stories: incident, trade-off, influence/adoption, failure/learning.
5. Final week: mock interviews with a friend who interrupts with "what if…".
