# 48 · Case Study: Zero-Downtime Deployments

> **Level:** ★★★ advanced
>
> **You will practise:** designing a deployment system so **releases never cause downtime** and bad releases are caught and reverted automatically: **graceful shutdown**, **progressive delivery** with **automated analysis**, **database migrations** (expand/contract), **feature flags**, **long-lived connections**, **stateful components**, **multi-region waves**, **rollback**, and measuring success with DORA metrics.
>
> **You need:** topics 06, 09, 22, 24, 28, 33–35.
>
> **Part 9 · Case studies** · Previous: [47 · CI/CD platform](47-case-cicd-platform.md) · Next: [49 · Central logging](49-case-central-logging.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Our company has 60 microservices on Kubernetes. Every deploy causes some errors and roughly once a month a deploy causes an outage. Design a deployment system with zero downtime."

| Question | Assumed answer |
|---|---|
| What does "downtime" mean here? | Any user-visible errors or latency SLO breach caused by a deploy |
| Deploy frequency? | ~80 deploys/day across services; want more |
| Traffic? | 10,000 req/s total; some services low-traffic |
| Protocols? | REST, gRPC internally, WebSockets for notifications, Kafka consumers |
| Databases? | Postgres per service (Aurora), some shared Redis |
| Regions? | 2 regions active-active |
| Current tooling? | GitHub Actions, Helm via `helm upgrade` from CI, rolling updates |
| Success criteria? | No deploy-caused SLO breaches; change failure rate < 5%; time to restore < 10 min |

---

## 2. Estimation

```
80 deploys/day → ~4 deploys/hour in working hours; each canary ~25 min → ~2–4 rollouts in flight at any time
Canary statistical power: a service with 500 req/s → 5% canary = 25 req/s → 5-min step = 7,500 requests
   → can detect an error-rate increase of ~0.5 pp reliably
Low-traffic service (5 req/s): 5% = 0.25 req/s → 75 requests in 5 min → too few → use longer steps, synthetic traffic, or blue/green with smoke tests
Current cost of failures: 1 outage/month × ~20 min × 10,000 req/s × ~50% errors ≈ 6 M failed requests
   vs SLO 99.9% of ~26 B requests/month = 26 M budget → one outage burns ~23% of the monthly budget
```

---

## 3. High-level design

```
Developer merge ─▶ CI (build once, sign, SBOM) ─▶ ECR
          │
          ▼
GitOps repo: envs/<env>/<region>/<service> (digest + rollout config)
          │ promotion bot (dev → staging → prod-region-1 → prod-region-2)
          ▼
Argo CD (per cluster) ─▶ Argo Rollouts (canary / blue-green) ─▶ traffic split via Gateway API / mesh
                                  │ AnalysisRuns
                                  ▼
                         Prometheus (SLIs, canary vs stable) ── abort ──▶ rollback + Slack/PagerDuty
Feature flag service (topic 57) ─ decouples release of behaviour from deploys
Migration Jobs (PreSync hooks) ─ expand/contract only, linted in CI
Deploy guardrails: error budget gate, freeze calendar, region waves with bake time
```

**Flow of one release:**
1. Merge → image built and signed.
2. Promotion bot commits digest to `dev` → Argo syncs → smoke tests → promote to staging (auto on green).
3. Prod region 1: PR auto-created; checks error budget > 25% and no freeze → merged.
4. Argo CD runs PreSync migration Job (expand-only), then Argo Rollouts starts canary: 5% → analysis 5 min → 25% → 10 min → 50% → 10 min → 100%.
5. Bake 30 min in region 1, then same in region 2.
6. Any failed analysis: instant traffic shift back to stable, rollout marked degraded, Git PR comment + team alert.

---

## 4. Deep dives

### 4.1 No errors during normal rollouts (pod lifecycle)
```yaml
spec:
  strategy: (Argo Rollouts canary with maxSurge 25%, maxUnavailable 0)
  template:
    spec:
      terminationGracePeriodSeconds: 45
      containers:
      - readinessProbe: { httpGet: { path: /readyz, port: 8080 }, periodSeconds: 5 }
        lifecycle:
          preStop: { exec: { command: ["sleep", "10"] } }   # let endpoints/LB update
# App: on SIGTERM stop accepting, drain in-flight (≤ 30 s), close DB pools, exit 0
```
+ ALB/Gateway deregistration delay 30 s, PDBs, HTTP keep-alive timeout > LB idle timeout (topics 06, 09).

### 4.2 Automated canary analysis
- Metrics compared **canary vs stable** (not absolute): 5xx ratio, p99 latency, saturation (CPU throttling, restarts), plus one **business metric** per critical service (e.g. payment authorization rate).
- Minimum sample size per step; if not reached, extend the step automatically.
- **Sticky canary** by user hash for user-facing services so a user doesn't flip between versions.
- For low-traffic services: blue/green with automated smoke + synthetic traffic against the preview service before switching.

### 4.3 Compatibility rules (enforced in review/CI)
- **APIs:** additive changes only; contract tests between consumers and providers.
- **Events (Kafka):** schema registry with backward-compatibility mode; consumers tolerate unknown fields.
- **DB:** expand/contract; migrations linted (no destructive DDL with code changes); `lock_timeout` set (topic 35).
- **Config:** validated against schemas; config-only changes go through the same canary.

### 4.4 Special workloads
| Workload | Strategy |
|---|---|
| WebSocket gateway | New pods for new connections; old pods send "reconnect" with jitter over 5–10 min before termination; long `terminationGracePeriodSeconds` |
| gRPC services | Mesh/L7 traffic split; `max_connection_age` on servers so clients rebalance |
| Kafka consumers | Rolling with cooperative rebalancing; canary as a small replica share; idempotent processing; watch DLQ rate and lag in analysis |
| Cron/batch | Version switch between runs; idempotent jobs |
| Stateful (Redis, operators) | Managed services or operator-driven rolling with replication checks; separate change process |

### 4.5 Feature flags
Risky behaviour ships dark; release by % with the flag service; kill switch per feature; flags have owners and expiry dates.

---

## 5. Scaling the process
- More deploys/day is safe because each is small and automated; queue promotions per service (Argo handles concurrency per app).
- Templated `Rollout` + `AnalysisTemplate` in the platform Helm chart so all 60 services get it by default (topic 39).
- Org-wide controls: deploy freeze switch (one flag in the promotion bot), error-budget gate.

---

## 6. Reliability and DR
- **Rollback paths:** automatic abort (seconds); `git revert` of the promotion commit (minutes); Argo Rollouts `abort`/`undo` from CLI if Git/CI unavailable (break-glass).
- **Keep previous images** (ECR lifecycle keeps last 30 tagged releases).
- **Region waves** ensure one region always runs the known-good version → if region 1 canary misses a bug that appears after an hour, region 2 is unaffected and can take traffic.
- **Rollback tested:** monthly game day where a deliberately bad version is deployed to staging and must be auto-aborted.

---

## 7. Security
- Only signed images from CI can run (admission policy); promotion bot uses a GitHub App with access only to the GitOps repo.
- Prod promotions require CODEOWNER approval for high-risk services or happen automatically for low-risk ones with good analysis history.
- Audit: each rollout ↔ commit ↔ digest ↔ approver.

---

## 8. Observability
- Deploy annotations on all dashboards.
- Rollout dashboard: in-flight rollouts, analysis results, aborts per service.
- **DORA:** deployment frequency, lead time, change failure rate (aborted + rolled back + incident-linked), time to restore.
- Alerts: rollout stuck > 1 h (ticket), aborted canary (team notification), SLO burn during rollout (page + auto-abort already).

---

## 9. Cost
- Canary adds a few extra pods temporarily (surge 25%) → negligible.
- Blue/green for low-traffic services doubles their pods briefly → small.
- Mesh/Gateway traffic splitting: if adopting a mesh only for this, consider Gateway API weights instead (cheaper, simpler).
- Engineering time saved by fewer incidents dwarfs infra cost.

---

## 10. Trade-offs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Canary by default | Smallest blast radius | Blue/green | Enough traffic for statistics on most services |
| Blue/green for low-traffic | Reliable verification | Canary | Too few requests for canary stats |
| Gateway API weights | Simple traffic split | Full mesh | Avoid mesh complexity unless needed for mTLS |
| Region waves + bake | Catch slow-burn bugs | Simultaneous | Slower lead time to full rollout (~1.5 h) accepted |

---

## 11. Follow-up questions

1. **Q: The canary passed but the bug showed up 3 hours later (memory leak). How do you catch that?**
   A: Bake times between waves, saturation metrics (memory growth rate) in post-rollout monitoring with auto-rollback on SLO burn, soak tests in staging for releases with risky changes, and region/cell waves so the blast radius stays limited.

2. **Q: How do you deploy a breaking API change?**
   A: You don't, directly. Add the new version alongside (`/v2` or new fields), migrate consumers, track old-version usage, then remove the old path after usage hits zero. Same expand/contract pattern as databases.

3. **Q: A migration must rewrite a 2 TB table. How does that fit zero-downtime deploys?**
   A: Decouple it: expand schema, deploy dual-write code, run a throttled background backfill over hours or days, switch reads behind a flag, and contract weeks later. The deploy pipeline only runs fast, lock-safe DDL.

4. **Q: How do you measure if the new system works?**
   A: Change failure rate, number of deploy-caused SLO breaches (target zero), time to restore (auto-abort should make it minutes), deployment frequency and lead time (should rise as confidence grows), and the error budget consumed by deploys.
