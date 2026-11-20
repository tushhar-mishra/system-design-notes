# 36 · IaC at Scale

> **Level:** ★★★ advanced
>
> **You will learn:** how to run Infrastructure as Code for **many teams, accounts and environments**: **state design and blast radius** (how to split state), **layering** (foundation → platform → app), **module strategy** (versioned, opinionated modules, a private registry), **multi-account landing zones**, **environments** (folders vs workspaces vs stacks), **pipelines for IaC** (plan on PR, apply on merge, one runner per state, OIDC), **policy as code**, **drift detection**, the **Terraform → Ansible handoff**, and the failure modes (state corruption, lock contention, provider upgrades, giant plans).
>
> **You need:** topics 08, 26, 32–33. Your Terraform notes are the foundation: [Terraform 34 · state deep dive](../../Terraform/notes/34-state-deep-dive.md), [Terraform 49 · environments and layout](../../Terraform/notes/49-environments-and-layout.md), [Terraform 63 · at scale](../../Terraform/notes/63-terraform-at-scale.md).
>
> **Part 7 · Infrastructure and platform** · Previous: [35 · DB changes](35-zero-downtime-database-changes.md) · Next: [37 · Config and secrets at scale](37-config-and-secrets-at-scale.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** One big Terraform state for everything works for a week. At scale you **split state by blast radius and ownership**, **layer** infrastructure by change frequency, **share** it through versioned modules, and run every change through a **pipeline with policy checks**.

```
Layer 0  org/accounts   (AWS Organizations, SCPs, SSO)          changes: rarely   owner: cloud platform
Layer 1  network        (VPC, TGW, DNS zones)                    changes: monthly  owner: network team
Layer 2  platform       (EKS clusters, shared DBs, registries)   changes: weekly   owner: platform team
Layer 3  apps           (per-service queues, buckets, IAM roles) changes: daily    owner: app teams
Each layer × each environment × each region = its own state.  Later layers READ earlier ones' outputs.
```

**Picture it.** A city. The land registry (accounts), roads and pipes (network), public buildings (platform), and shops (apps) are managed by different departments with different permits. A shop renovation can't accidentally dig up a water main, because they're in different plans (states) with different owners.

**The one thing to know.** **Blast radius = what a single `apply` can break.** Small states, owned by one team, applied by a pipeline (not laptops), with plan review and policy checks, are how large orgs run Terraform safely.

**Try it now:** `terraform state list | wc -l` on your biggest state. Hundreds of resources from several teams in one state = a blast-radius problem.

**You've got it if you can:** design a state layout for a multi-account org, describe the IaC pipeline, and explain module versioning and drift handling.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Concept | Meaning |
|---|---|
| **IaC** (Infrastructure as Code) | Infrastructure defined in code, versioned, reviewed, applied by tools ([Terraform 01](../../Terraform/notes/01-what-is-iac.md)) |
| **State** | Terraform's record mapping code to real resources ([Terraform 12](../../Terraform/notes/12-state-basics.md)) |
| **Root module / stack** | A directory you `apply`, with its own state |
| **Module** | Reusable building block (a VPC, an EKS cluster, a "service" bundle) |
| **Landing zone** | Pre-built multi-account foundation: accounts, guardrails, networking, logging, identity (AWS Control Tower / Landing Zone Accelerator, Azure Landing Zones, GCP landing zone blueprints) |
| **SCP** (Service Control Policy) | Org-level guardrail restricting what any account can do (e.g. deny leaving approved regions) |
| **Policy as code** | Automated rules on plans/configs (OPA/Conftest, Sentinel, Checkov, tfsec/Trivy) ([Terraform 54](../../Terraform/notes/54-policy-as-code.md)) |
| **Drift** | Real infrastructure differs from code ([Terraform 37](../../Terraform/notes/37-drift.md)) |
| **TACOS** | "Terraform Automation and Collaboration Software": HCP Terraform/Terraform Enterprise, Spacelift, env0, Scalr, Atlantis (OSS) |

---

## Why it exists

Problems at scale:
- **Giant states:** 2,000 resources → 10-minute plans, lock contention (everyone waits), one mistake can destroy unrelated things.
- **Copy-paste infrastructure:** 50 slightly different VPCs, impossible to patch consistently.
- **Laptop applies:** no audit trail, wrong credentials, local state, "it worked on my machine".
- **No guardrails:** public buckets, unencrypted DBs, `0.0.0.0/0` security groups reach prod.
- **Drift:** console clicks during incidents never reconciled.

---

## Analogy

The city departments (Core). **Modules** are standard building codes and pre-approved blueprints: every new shop uses the approved electrical design, so safety is built in. **Policy as code** is the building inspector who checks every plan before construction.

---

## How it works inside

### State design: how to split

Split by:
1. **Environment** (dev/staging/prod): never share state across environments.
2. **Account and region.**
3. **Layer / lifecycle** (network vs cluster vs app): things that change together live together.
4. **Ownership** (team): one owner per state.
5. **Blast radius:** a state whose loss/misapply would be catastrophic (DNS zones, network core) is kept small and tightly controlled.

Rough size guide: tens to a few hundred resources per state; plans under ~2–3 minutes.

```
infra-live/
├─ org/                        (management account: OUs, SCPs, Identity Center)
├─ shared/network/ap-south-1/  (TGW, shared DNS)
├─ prod/
│  ├─ ap-south-1/
│  │  ├─ network/              state: prod/ap-south-1/network
│  │  ├─ eks-platform/         state: prod/ap-south-1/eks-platform
│  │  └─ data/                 state: prod/ap-south-1/data (shared Aurora)
├─ staging/... dev/...
service repos (app teams):  orders/infra/{dev,staging,prod}/  → SQS, S3, IAM role for the service
```

**Sharing outputs between states:** `terraform_remote_state` (tight coupling: reader needs access to the whole state) or better **published outputs** in SSM Parameter Store / a data source lookup by tags ([Terraform 38 · sharing state](../../Terraform/notes/38-sharing-state.md)). Avoid deep chains of dependencies.

**Environments:** folders per environment (explicit, different backends/credentials) are the common choice for prod-grade setups; **workspaces** are fine for identical ephemeral copies (feature environments) but share backend config and make "which env am I in?" mistakes easier ([Terraform 48](../../Terraform/notes/48-workspaces.md)). Tools like Terragrunt or Terraform Stacks reduce folder duplication.

### Module strategy

```
Level 1: resource modules      (thin wrappers: s3-bucket with encryption + versioning + block public access ON)
Level 2: component modules     (eks-cluster, vpc, rds-postgres with backups/Multi-AZ/alarms)
Level 3: service/"product" modules (a "standard-service" = IAM role + SQS + S3 + dashboards + alarms)
```
- **Opinionated defaults** = secure by default (encryption, logging, tags, no public access unless explicitly justified).
- **Versioned** with SemVer tags, published in a **private registry**; consumers pin `version = "~> 3.2"` ([Terraform 45](../../Terraform/notes/45-module-sources-and-versioning.md)).
- **Tested:** `terraform test`, policy checks, examples applied in a sandbox account in CI ([Terraform 55](../../Terraform/notes/55-testing.md)).
- **Upgrades:** changelogs, `moved` blocks to avoid resource replacement, deprecation windows. Bots (Renovate) open upgrade PRs across repos.
- ⚠️ Avoid "mega-modules" with 80 input variables that try to cover every case; they become their own maintenance burden.

### The IaC pipeline

```
PR opened
  → fmt / validate / tflint
  → security scan (Trivy/Checkov) + policy (OPA/Conftest or Sentinel) on code
  → terraform plan (per affected state, using a read-only OIDC role)
  → policy on the PLAN JSON (e.g. "no deletes of DBs", "only approved instance types", cost estimate via Infracost)
  → plan posted as PR comment; reviewers see exactly what will change
Merge (after approval from CODEOWNERS of that state)
  → terraform apply of the SAVED plan (apply role, environment protection for prod)
  → one apply per state at a time (concurrency group / lock)
  → notifications, audit log
Scheduled: drift detection (plan -detailed-exitcode nightly) → ticket on drift
```
([Terraform 57 · GitHub Actions](../../Terraform/notes/57-github-actions.md), [Terraform 58 · Jenkins](../../Terraform/notes/58-jenkins.md), [Terraform 56 · Git workflow](../../Terraform/notes/56-git-workflow.md).)

- **Separate roles:** `plan` role is read-only; `apply` role is scoped to the account/state; prod apply needs approval.
- **Humans don't have write access to prod in the console** by default; break-glass roles with alerting and session recording.
- **Applying the exact reviewed plan** (`terraform apply plan.tfplan`) avoids "the plan changed between review and apply".

### Multi-account landing zone

```
Management account (Organizations, billing, SCPs)  ← almost nothing runs here
├─ Security OU: log-archive account (CloudTrail org trail, Config), security-tooling (GuardDuty admin)
├─ Infrastructure OU: network (TGW, DNS), shared-services (CI runners, registry, monitoring)
├─ Workloads OU: prod OU (team-a-prod, team-b-prod), non-prod OU (…-dev, …-staging)
└─ Sandbox OU: experiments, budgets, auto-cleanup
Guardrails: SCPs (deny disabling CloudTrail, deny non-approved regions, deny root use), IAM Identity Center for human access
Account vending: new accounts created via code (Account Factory for Terraform / Control Tower) with baseline applied
```
Why many accounts: **hard blast-radius and security boundaries**, clear cost attribution, per-account service quotas. GCP: folders/projects; Azure: management groups/subscriptions.

### Terraform + Ansible (and images)

```
Packer (+ Ansible) → golden AMI (baseline hardening, agents) ── weekly, scanned
Terraform → VPC, ASG/launch template using the AMI, LB, DNS, IAM
cloud-init → small first-boot bits (join cluster, fetch config)
Ansible → day-2 config for long-lived VMs (or ansible-pull), inventory from cloud tags
Kubernetes workloads → GitOps (topic 33), not Terraform per deployment
```
([Ansible 43 · Ansible and Terraform](../../Ansible/notes/43-ansible-and-terraform.md), [Linux 104 · cloud-init](../../linux-mastery-rhel/notes/104-cloud-init.md).) Rule: **immutable where possible** (bake images, replace instances), configuration management where instances must live long.

### Drift

- Detect: scheduled `plan` per state (exit code 2 = changes) → ticket/Slack; AWS Config rules for critical settings.
- Respond: either **revert** (apply code) or **adopt** (update code to match, `import` new things, [Terraform 39](../../Terraform/notes/39-import.md)).
- Prevent: no console write access; incident changes must be codified within 24 h.

---

## Options and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| Orchestration | Plain CI pipelines / Atlantis / HCP Terraform / Spacelift / env0 | CI pipelines are fine with discipline; TACOS add policy, drift, RBAC, run queues out of the box |
| Tool | Terraform / OpenTofu / Pulumi / CloudFormation-CDK / Crossplane | Terraform/OpenTofu most common ([Terraform 04](../../Terraform/notes/04-license-change-and-opentofu.md)); Crossplane for K8s-native control plane IaC |
| Environments | Folders / workspaces / Terragrunt / Stacks | Folders for long-lived envs; workspaces for ephemeral copies |
| Module source | Monorepo modules / separate repos + registry | Registry + SemVer at scale |
| App team autonomy | Central team writes all IaC / app teams own app-layer IaC with modules and policy | Self-service with guardrails (topic 39, 53) |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Giant state** | Slow plans, lock contention, scary applies | Plan time, resources per state | Split by layer/env/owner ([Terraform 40 · moved/removed](../../Terraform/notes/40-moved-and-removed.md) to refactor) |
| **Apply destroys something critical** | Outage, data loss | Plan review missed `destroy` | Policy on plan ("deny delete of aws_db_instance in prod"), `prevent_destroy`, deletion protection on resources |
| **State corruption / lost state** | Terraform wants to recreate everything | Plan shows all creates | Versioned encrypted state bucket, backups, recovery runbook ([Terraform 42](../../Terraform/notes/42-state-recovery.md)) |
| **Stale lock** | All applies blocked | Lock error | CI concurrency, force-unlock checklist ([Terraform 36](../../Terraform/notes/36-state-locking.md)) |
| **Provider upgrade changes behaviour** | Unexpected diffs | Plan after upgrade | Pin providers, lock file, upgrade via PRs with plans per state ([Terraform 62](../../Terraform/notes/62-upgrades.md)) |
| **Module change ripples everywhere** | Many states replanning with changes | Mass diffs | SemVer, canary-upgrade a few states first |
| **Cross-state dependency cycles** | Can't create from scratch | DR rebuild test fails | Clear layering; test full rebuild in a sandbox |
| **Secrets in state** | State readers see DB passwords | Audit | Encrypt state, restrict access, generate secrets outside TF or use managed secret rotation ([Terraform 51](../../Terraform/notes/51-secrets.md)) |
| **Cloud API throttling** | Applies fail with `Throttling` | Errors | Lower `-parallelism`, smaller states, retries |

**3 AM:** "A Terraform apply in prod deleted the DB security group rule; app can't reach DB." Mitigate: re-apply the previous known-good commit (or restore the rule manually via break-glass and codify it). Postmortem: plan policy for SG rule deletions in prod, two-person review on network/data layers.

---

## Where you'll use it

- **Case study 53** (self-service infrastructure for 50 teams).
- **Every cloud design**: say how it's built (Terraform states/modules) and changed (pipeline).
- **Terraform capstones** ([Terraform 59–61](../../Terraform/notes/59-capstone-1-foundation.md)).
- **DR**: infrastructure rebuild time is part of RTO (topic 41).

---

## Design exercise

**Prompt:** A company has one Terraform repo with one state per environment (dev/prod), ~1,800 resources in prod, applied from engineers' laptops with admin keys. 12 teams now want to self-serve. Propose the target.

<details>
<summary><b>Model answer</b></summary>

1. **Accounts:** move to a multi-account landing zone: org/security/network/shared accounts + per-team prod and non-prod accounts (or per domain), via account vending.
2. **States:** split prod into layers per account/region: `network`, `platform` (EKS, shared DBs), `observability`, and per-team `app` states. Refactor with `moved`/`import`/`removed` blocks, not by recreating resources. Target < 200 resources per state.
3. **Modules:** private registry with versioned opinionated modules (vpc, eks, rds, s3, standard-service). Security defaults baked in.
4. **Pipeline:** Atlantis or HCP Terraform/Spacelift (or GitHub Actions) with plan on PR, policy checks on plan (OPA: no public S3, encryption on, no deletes of stateful resources in prod without a label), Infracost comments, CODEOWNERS approvals per state, apply on merge using **OIDC** roles per account. One run per state at a time.
5. **Access:** remove admin keys; humans get read-only via SSO; break-glass role with alerts.
6. **Drift:** nightly plans; drift tickets to the owning team.
7. **Ownership:** platform team owns layers 0–2 and modules; app teams own their app states using modules.
8. **Migration plan:** freeze laptop applies → stand up pipeline on current states → split network first (most critical, most stable) → then platform → then teams onboard one by one.

Trade-offs: more states mean more cross-state dependencies to manage (published outputs), more pipeline runs, and a learning curve, in exchange for small blast radius, parallel work, and audited changes.
</details>

---

## Quick recap

- Split state by environment, account/region, layer and owner; keep each small (blast radius).
- Layer infra by change frequency; share outputs via published parameters, not deep remote-state chains.
- Versioned, opinionated, tested modules in a private registry.
- Pipeline: plan on PR + policy on plan + review → apply exact plan on merge, via OIDC, one run per state.
- Multi-account landing zone with SCP guardrails; detect drift nightly; bake images, configure long-lived VMs with Ansible.

### Interview questions

1. **Q: How do you decide how to split Terraform state?**
   A: By blast radius and ownership: separate environments, accounts and regions; separate layers that change at different rates (network, platform, apps); one owning team per state; keep each to tens or a few hundred resources so plans are fast and a mistake is contained. Share outputs through published parameters or data sources.

2. **Q: How do you stop a Terraform change from accidentally deleting a production database?**
   A: Several layers: `deletion_protection` on the DB itself, `prevent_destroy` lifecycle in code, policy as code on the plan JSON that blocks destroys/replacements of stateful resources in prod, mandatory plan review by owners, applying only the reviewed plan, and backups/PITR as the last line.

3. **Q: Workspaces or directories for environments?**
   A: Directories for long-lived environments: explicit backends, credentials and variables per env, and different code versions can be promoted. Workspaces suit identical, short-lived copies (per-PR environments) where the same code and backend apply; for prod they make it too easy to apply to the wrong env.
