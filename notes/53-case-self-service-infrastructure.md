# 53 · Case Study: Self-Service Infrastructure for 50 Teams

> **Level:** ★★★ advanced
>
> **You will practise:** designing a **self-service infrastructure platform** on Terraform: the **request interface** (PRs, portal, or Kubernetes-style claims), **opinionated versioned modules**, **state and account layout**, **plan/apply pipeline** with **policy as code**, **approvals by risk**, **cost estimates**, **drift detection**, **module upgrades at scale**, ownership, and failure handling (stuck applies, orphaned resources).
>
> **You need:** topics 22, 32, 36–40, 43.
>
> **Part 9 · Case studies** · Previous: [52 · Database DR](52-case-database-dr.md) · Next: [54 · Secrets management service](54-case-secrets-management.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "50 product teams wait 1–2 weeks for the central infra team to create databases, queues, buckets and IAM roles. Design self-service infrastructure."

| Question | Assumed answer |
|---|---|
| What do teams request? | Postgres/MySQL (RDS/Aurora), Redis, SQS/SNS, S3 buckets, IAM roles for services, DNS records, Kafka topics, K8s namespaces |
| Cloud/IaC? | AWS, multi-account (per team prod/non-prod), Terraform, GitHub, GitHub Actions |
| Volume? | ~200 changes/week, growing |
| Guardrails? | Encryption, no public access, tagging, approved instance sizes, cost limits, prod changes reviewed |
| Speed goal? | Non-prod in < 30 min self-service; prod in < 1 day with appropriate review |
| Audit? | SOC 2: who changed what, approvals |
| Skills? | Most product engineers know a little Terraform; some know none |

---

## 2. Estimation

```
200 changes/week ≈ 40/day; each = plan (2 min) + apply (2–20 min; RDS ~15 min)
Pipeline concurrency: ~40/day over 8 h → 5/h → Little's Law: 5/h × 0.25 h ≈ 1–2 concurrent → small
States: 50 teams × 2 envs × ~3 components ≈ 300 states (+ platform states) → plan/apply automation must be per state
Policy evaluation: per plan, seconds
Drift detection: 300 states nightly × 2 min plan ≈ 10 runner-hours/night
```
**Conclusion:** throughput is easy; the hard parts are **safe defaults, guardrails, ownership, approvals that don't become the new bottleneck, and lifecycle (upgrades, drift, deletion)**.

---

## 3. High-level design

```
Interface options (pick per audience):
  A) Git PR with a small "stack" file (most teams)      B) Portal form (Backstage template) → generates the same PR
  teams/payments/prod/infra.yaml:
     postgres: { name: orders-db, size: medium, ha: true }
     queues:   [ { name: order-events, dlq: true } ]
     buckets:  [ { name: invoices, retention_days: 2555 } ]
                │
                ▼
GitHub repo "infra-live" (or per-team repos) ─ PR ─▶ Pipeline (GitHub Actions / Atlantis / TACOS)
   1. render: YAML → Terraform root module using PLATFORM MODULES (pinned versions)
   2. fmt/validate/tflint + Checkov/Trivy
   3. plan with read-only OIDC role in the team's account
   4. policy on plan JSON (OPA/Conftest): encryption, no public, tags, allowed sizes/regions,
      no destroy of stateful resources without label, cost delta (Infracost) ≤ budget
   5. post plan + policy result + cost to PR
   6. approval rules by risk: non-prod low-risk → auto-merge; prod or cost > $X/month or destroy → CODEOWNERS (team lead + platform)
   7. merge → apply SAVED plan with apply role (OIDC, scoped to team account) → one run per state at a time
   8. outputs published (SSM parameters / Secrets Manager / service catalog), notifications
Nightly: drift detection per state → ticket to owning team ; Renovate PRs for module upgrades
Backstage catalog: each resource linked to owner, cost, dashboards
```

---

## 4. Deep dives

### 4.1 Interface: PR-based vs portal vs Crossplane
| Interface | Pros | Cons |
|---|---|---|
| **PR with simple YAML → generated Terraform** ✅ | Review, audit, Git history; no TF expertise needed; escape hatch to raw TF | Generator to maintain |
| Raw Terraform with modules | Full flexibility | Requires TF skills; more ways to go wrong |
| Portal (Backstage scaffolder) | Friendly for non-TF users | Should still produce PRs (single source of truth) |
| Crossplane claims (`kind: PostgresInstance`) in K8s/GitOps | K8s-native, continuous reconciliation | Another control plane; teams must know K8s; harder plan previews |
| Ticket to central team | Expert-reviewed | The bottleneck we're removing |

### 4.2 Modules (the product)
- `postgres` module: Aurora/RDS with Multi-AZ in prod (forced), encryption with team CMK, backups (PITR 7–35 d by tier), deletion protection in prod, parameter group defaults, CloudWatch alarms + dashboard, Secrets Manager-managed master password, `t-shirt` sizes (small/medium/large → instance classes).
- `queue` module: SQS + DLQ + redrive + alarms on age/DLQ depth.
- `bucket` module: Block Public Access, versioning, SSE-KMS, lifecycle by retention, access logging.
- `service-identity` module: IAM role for a K8s ServiceAccount (Pod Identity) with **generated least-privilege policy** from declared resources (role may only touch its own queues/buckets).
- SemVer, changelogs, `terraform test`, example stacks applied in a sandbox account in CI.

### 4.3 State and accounts
- One state per team × env × component (e.g. `payments/prod/data`), stored in a per-account S3 backend with lock files.
- Team accounts created by account vending (topic 36) with SCPs, budgets, baseline (CloudTrail, Config, GuardDuty).

### 4.4 Approvals by risk (keep humans for what matters)
| Change | Approval |
|---|---|
| Non-prod create/update, policy-clean, cost < $500/month | Auto |
| Prod create/update, policy-clean | Team lead (CODEOWNERS) |
| Any destroy/replace of stateful resource | Team lead + platform engineer + label `allow-destroy` |
| Policy exception | Platform/security review, time-limited exception record |
| Cost increase > $5k/month | + FinOps/budget owner |

### 4.5 Lifecycle and failures
- **Apply failure mid-way:** Terraform state reflects what was created; pipeline posts the error; re-run converges (idempotent). Stuck locks: pipeline concurrency + force-unlock runbook.
- **Orphans:** resources created outside IaC detected by tag scans (`managed-by != terraform`) → ticket; sandbox auto-cleanup.
- **Module upgrades:** Renovate PRs per stack with plan output; breaking changes require `moved` blocks; deprecation dashboard showing stacks on old versions.
- **Decommissioning:** a "delete" PR with destroy approvals; final snapshot for DBs; retention of backups per policy.

---

## 5. Scaling
- Per-team repos or monorepo with path-based triggers; plans only for changed stacks.
- Parallel runs across states; serialized per state.
- Policy library versioned and tested (OPA unit tests).
- Self-service of new resource types: platform adds a module + schema to the YAML generator.

---

## 6. Reliability and DR
- Pipeline is on the path for infra changes, not for running apps; still: break-glass procedure (local apply with elevated role, recorded) for incidents.
- State buckets versioned + replicated; state recovery runbook ([Terraform 42](../../Terraform/notes/42-state-recovery.md)).
- Full-rebuild test: a sandbox account rebuilt from code quarterly (proves IaC completeness for DR, topic 41).

---

## 7. Security
- OIDC roles: plan (read-only) vs apply (write, scoped to that team's account); prod apply only from `main` after approval (environment protection).
- No one has console write access in prod accounts by default.
- Policies prevent privilege escalation (no `iam:*` grants, permission boundaries required on created roles).
- Audit: PR + approvals + pipeline logs + CloudTrail.

---

## 8. Observability
- Platform metrics: time from PR open to applied (p50/p90), auto-approval rate, policy failure reasons (which rules block teams most → improve docs/modules), apply failure rate, drift count by team.
- Resource dashboards and alarms come **with** each module (teams get monitoring for free).
- Cost per team (tags enforced by modules + account-level allocation).

---

## 9. Cost
- Platform cost: runners, TACOS license (if bought), engineering team.
- Savings: fewer tickets (central team time), right-sized defaults (t-shirt sizes), lifecycle rules, budget gates catching expensive mistakes in PRs (Infracost).

---

## 10. Trade-offs

| Decision | Chosen | Gave up |
|---|---|---|
| YAML abstraction over modules | Easy for most teams | Some flexibility → escape hatch: raw TF with modules in the same pipeline |
| Policy on plan | Catches real changes (not just code patterns) | Plans must run first; slower feedback than static checks alone → do both |
| Risk-based approvals | Speed for low-risk | Requires trust in policies and good tests |
| PR-based vs Crossplane | Plan preview and familiar review | No continuous reconciliation → nightly drift detection compensates |

---

## 11. Follow-up questions

1. **Q: How do you stop a team from creating a $50k/month database by mistake?**
   A: T-shirt sizes in modules (no free-form instance classes), Infracost delta in the PR with a policy threshold requiring budget-owner approval, account budgets with alerts, and SCPs denying very large instance families except via exception.

2. **Q: A team needs something the modules don't support. What happens?**
   A: Escape hatch: they write raw Terraform in their stack using allowed providers; the same policies apply. If several teams need it, the platform adds it to a module. Policy exceptions are time-limited and recorded.

3. **Q: How do you upgrade the `postgres` module across 120 stacks?**
   A: Release a new SemVer version with changelog and `moved` blocks; Renovate opens PRs per stack showing plans; non-prod auto-merges if the plan has no replacements; prod requires approval; a dashboard tracks adoption; the old major version gets a deprecation date.

4. **Q: Why not let teams just use the AWS console in their own accounts?**
   A: No review, no audit trail of intent, no consistent security defaults, drift and snowflakes, hard to reproduce for DR. Console read access is fine; changes go through code with guardrails, which is faster in the long run because it's automated and repeatable.
