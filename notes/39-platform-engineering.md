# 39 · Platform Engineering and Internal Developer Platforms

> **Level:** ★★★ advanced
>
> **You will learn:** what **platform engineering** and an **IDP** (Internal Developer Platform) are, **golden paths** and **paved roads** vs **guardrails**, the parts of a platform (**developer portal, service catalog, templates/scaffolding, self-service infrastructure, delivery, observability, security**), **platform as a product** (users, roadmap, adoption, SLOs), **team topologies**, measuring with **DORA** and **developer experience** metrics, and the common failure modes (ticket-ops, mandatory-but-bad platforms, over-abstraction).
>
> **You need:** topics 32–38.
>
> **Part 7 · Infrastructure and platform** · Previous: [38 · Kubernetes platform design](38-kubernetes-platform-design.md) · Next: [40 · Security architecture](40-security-architecture.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Platform engineering** = building and running internal products (CI/CD, clusters, infrastructure modules, observability, security tooling) that let product teams **ship and operate software by themselves**, quickly and safely, without filing tickets.

```
Developer: "I need a new service with a DB and a queue"
   → Portal (Backstage): pick template "Java service + Postgres + SQS"
   → Scaffolder: creates repo (code skeleton, Dockerfile, pipeline using golden template, Helm chart)
                 + infra request (Terraform module PR) + namespace + dashboards + SLO + catalog entry
   → 30 minutes later: deployed to dev with monitoring, secrets, TLS, policies. No tickets.
```

**Picture it.** A city's road network. A **paved road** (golden path) is fast, signposted and maintained; you *can* go off-road, but you'll need your own 4×4 and repairs. **Guardrails** on the highway keep you safe without telling you where to drive.

**The one thing to know.** **Treat the platform as a product.** Its users are developers; adoption must be earned by being the easiest, safest path, not mandated. Measure it by developer outcomes (lead time, deploy frequency, time to first deploy), not by how many tools you installed.

**Try it now:** time how long it takes a new engineer at your company to deploy a "hello world" service to production with monitoring. That number is your platform's headline metric.

**You've got it if you can:** describe the IDP components, explain golden paths vs guardrails, and define how you'd measure a platform's success.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **Platform team** | Team building internal self-service capabilities for product teams |
| **IDP** (Internal Developer Platform) | The integrated set of tools, APIs and workflows that provide self-service |
| **Developer portal** | The UI/front door: catalog, docs, templates, scorecards (Backstage, Port, Cortex, OpsLevel) |
| **Service catalog** | Inventory of services with owner, on-call, repo, dependencies, SLOs, docs |
| **Golden path / paved road** | Opinionated, supported, documented way to do common tasks |
| **Guardrails** | Automated limits that keep any path safe (policies, quotas, required checks) |
| **Scorecards** | Per-service checks of standards (has SLO, has runbook, on supported base image) |
| **Cognitive load** | Mental effort developers spend on things outside their product |
| **Team Topologies** | Model of team types: stream-aligned (product), platform, enabling, complicated-subsystem |

---

## Why it exists

"You build it, you run it" plus cloud-native tooling gave product teams huge cognitive load: Kubernetes YAML, Terraform, IAM, pipelines, observability, security scanners, certificates... Each team re-solving these is slow, inconsistent and insecure. Central ops teams with ticket queues are the opposite extreme: safe but slow. Platform engineering aims for **both speed and safety** via self-service with built-in guardrails.

---

## Analogy

Paved roads and guardrails (Core). Also: a **restaurant franchise kit**: a new restaurant gets the kitchen layout, equipment, suppliers, recipes and health-code checklists ready-made, and focuses on cooking and customers. They can still add local dishes.

---

## How it works inside

### Reference architecture of an IDP

```
                         ┌──────────────────────────────────────────┐
Developers ─────────────▶│ Developer portal (Backstage)              │ catalog, docs (TechDocs), templates,
   (UI / CLI / Git)      │                                          │ scorecards, links to dashboards
                         └──────────────┬───────────────────────────┘
                                        │ self-service actions (create service, request DB, new env)
          ┌─────────────────────────────┼──────────────────────────────┐
          ▼                             ▼                              ▼
  Integration & delivery        Infrastructure orchestration     Observability & ops
  (CI templates, registry,      (Terraform modules + pipeline,   (dashboards, alerts, SLO templates,
   GitOps, progressive           Crossplane claims, account/      on-call integration, runbooks)
   delivery)  topics 32–34       namespace vending) topics 36–38   topics 27–30
          └─────────────────────────────┼──────────────────────────────┘
                                        ▼
            Security & governance (policies, secrets, identity, scanning, cost) topics 37, 40, 43
                                        ▼
                    Resources: cloud accounts, K8s clusters, DBs, queues
```

### Golden path example: "new service"

1. Developer picks a template in the portal and fills a short form (name, team, language, needs DB?).
2. Scaffolder creates the repo with: app skeleton (health endpoints, OTel instrumentation, structured logging), Dockerfile (approved base image), CI pipeline referencing the versioned golden template, Helm/Kustomize manifests, `catalog-info.yaml` (owner, tier, SLO).
3. It opens PRs to: GitOps repo (dev namespace + app), infra repo (DB via standard module), alerting repo (SLO-based alerts from template).
4. Pipelines run; service appears in the catalog with links to dashboards, logs, traces, runbook template.
5. Day 2: the platform team upgrades templates; Renovate-style bots open upgrade PRs to all services.

### Abstraction levels: how much to hide?

| Level | Developer writes | Pros | Cons |
|---|---|---|---|
| Raw cloud/K8s | Terraform + YAML from scratch | Full control | High load, inconsistency |
| **Modules/charts + templates** ✅ common | Small values file, module inputs | Flexible, familiar | Still some K8s/Terraform knowledge needed |
| **Platform API / abstraction** (Score, Crossplane compositions, KubeVela, internal CRDs like `kind: Service`) | "I need a web service with Postgres, size M" | Very low load | Abstraction must be well designed; escape hatches needed |
| PaaS (Heroku-like, Cloud Run, App Runner) | Just code | Lowest load | Least control; may not fit all workloads |

Senior answer: choose the **thinnest viable platform**: the smallest set of abstractions that removes most toil for most teams, with **escape hatches** for the 10–20% of special cases.

### Platform as a product

- **Users:** product engineers. Do discovery (interviews, surveys, shadowing), not assumptions.
- **Roadmap:** driven by user pain and company goals (security, cost, reliability).
- **Adoption:** voluntary where possible; make the golden path the easiest; migrate with help (enabling teams), not mandates.
- **Documentation and support:** docs, office hours, Slack channel, internal SLAs for support.
- **SLOs for platform components** (CI queue time, cluster API availability, deploy success) (topic 28).
- **Versioning and deprecation** policies so changes don't break teams.

### Measuring success

**DORA metrics** (DevOps Research and Assessment, Google's long-running research program):

| Metric | Meaning | Elite-ish direction |
|---|---|---|
| **Deployment frequency** | How often you deploy to prod | On demand, many times per day |
| **Lead time for changes** | Commit → running in prod | Less than a day |
| **Change failure rate** | % of deploys causing incidents/rollbacks | Low (single digit to ~15%) |
| **Time to restore service** (failed deployment recovery time) | Incident → recovered | Under an hour |
| (Reliability, added later) | Meeting operational goals | |

Plus platform-specific: time to first deploy for a new service/engineer, % of services on golden paths, scorecard compliance, developer satisfaction (survey, e.g. SPACE framework dimensions), support ticket volume, cost per service. ⚠️ Don't turn DORA metrics into team targets for comparison; they become gamed.

### Team topologies

```
Stream-aligned teams (product)  ── consume ──▶  Platform team(s) (X-as-a-Service)
        ▲                                             │
        └─────── Enabling team (temporary help: migrations, coaching) ◀─┘
Complicated-subsystem team (e.g. search engine, ML infra)
```
Interaction modes: **X-as-a-Service** (default), **collaboration** (temporary, to build something new together), **facilitating** (enabling team teaches).

---

## Options and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| Portal | Backstage (OSS, flexible, needs a team to run) / SaaS portals (Port, Cortex) / none (docs + Git) | Start with catalog + templates; a portal is a means, not the goal |
| Self-service infra | Terraform modules via PRs / Crossplane / TACOS no-code modules / service catalog products | PR-based with modules and policy is a robust start (case study 53) |
| Mandate vs earn | Mandatory golden path / optional | Mandate guardrails (security, compliance); earn adoption of paths |
| Build vs buy | Build integrations / buy PaaS | Buy commodity, build where your company is unique |

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Ticket-ops platform** | "Self-service" = filling a form that a human executes | Automate end to end; measure lead time of requests |
| **Ivory tower** | Platform built without users' input; low adoption | Product management, user research, internal customers on the roadmap |
| **Over-abstraction** | Teams blocked by missing features, fork the platform | Escape hatches, thinner abstractions, extensibility |
| **Portal without substance** | Pretty Backstage, stale catalog | Catalog generated from real sources (repos, cloud tags, K8s), scorecards with automation |
| **Breaking changes to templates** | Mass pipeline failures | Versioned templates, canary rollout, deprecation windows |
| **Platform as SPOF** | Platform outage blocks all deploys | Platform SLOs, HA, break-glass paths |
| **No ownership data** | Incidents with no known owner | Catalog ownership required in CI (no deploy without owner) |
| **Metrics theater** | Gamed DORA numbers | Use metrics for learning, not ranking |

---

## Where you'll use it

- **Senior/staff interviews** for platform roles: "What would you build first?" → the biggest developer pain with measurable outcome (often: CI speed, environment provisioning, or a golden path for new services).
- **Case studies 47 and 53** (CI/CD platform, self-service infra) are platform products.
- **Jenkins shared libraries / GHA reusable workflows** are early golden paths ([Jenkins 58](../../Jenkins/notes/58-shared-libraries-production.md), [GHA 22](../../GitHub-Actions-Basics-To-Production/notes/22-reusable-workflows.md)).
- **Ansible AAP** as a self-service automation portal ([Ansible 56](../../Ansible/notes/56-awx-and-aap.md)).

---

## Design exercise

**Prompt:** You join a 300-engineer company as the first platform lead (team of 6). Pain points from a survey: "new service takes 3 weeks to reach prod", "we don't know who owns what during incidents", "every team has different pipelines", "cloud costs are unknown". What's your 12-month plan?

<details>
<summary><b>Model answer</b></summary>

**Months 0–2: discovery and quick wins**
- Interview 10 teams, map the "new service" journey and its 3 weeks of waiting (tickets for repo, DNS, DB, namespace, monitoring, security review).
- Quick win: **service catalog** (Backstage or even a YAML-in-repo catalog) with **mandatory owner + on-call** for all services, generated/validated from repos; used immediately in incidents.
- Baseline metrics: time to first deploy, DORA metrics, CI duration, cost visibility.

**Months 2–6: golden path v1 for the most common stack (e.g. Java/Go service on EKS)**
- Versioned CI template (build, test, scan, sign, push) + GitOps deploy (topics 32–33).
- Scaffolder template creating repo + namespace + dashboards + alerts + catalog entry.
- Standard Terraform modules for Postgres/SQS/S3 via PR-based self-service with policy checks.
- Target: new service to prod in **< 1 day**.

**Months 6–9: adoption and guardrails**
- Enabling work: migrate 30% of services, prioritizing teams with most pain.
- Guardrails mandatory for all (signed images, policies, secrets via ESO), golden path optional but easiest.
- Cost: tagging/labels enforced → showback dashboards per team (topic 43).

**Months 9–12: second stack (e.g. Python/data jobs), scorecards, platform SLOs**
- Scorecards (has SLO, runbook, supported base image).
- Platform SLOs published (CI queue time, deploy success).
- Measure: time to first deploy, % on golden path, DORA trend, developer satisfaction survey.

Trade-off: we don't build a full abstraction layer or a heavy portal in year one; we remove the biggest waits first and earn adoption.
</details>

---

## Quick recap

- Platform engineering = internal self-service products that reduce developers' cognitive load with built-in safety.
- IDP: portal/catalog, templates, delivery, self-service infra, observability, security/governance.
- Golden paths are optional and easiest; guardrails are mandatory and automated; thinnest viable platform with escape hatches.
- Treat the platform as a product: users, roadmap, docs, support, SLOs, versioning.
- Measure with DORA, time to first deploy, adoption and satisfaction, never as a ranking tool.

### Interview questions

1. **Q: What's the difference between a golden path and a guardrail?**
   A: A golden path is a recommended, supported way to do something (create a service, deploy, add a DB), designed to be the easiest option. A guardrail is an automated constraint that applies to every path (signed images only, no public buckets, quotas). Paths drive adoption by convenience; guardrails enforce safety regardless of path.

2. **Q: How do you know your platform is succeeding?**
   A: Developer outcomes: time to first deploy for a new service, lead time and deploy frequency, change failure rate and restore time, adoption of golden paths, fewer tickets, satisfaction surveys, and platform SLOs being met. Not the number of tools or features shipped.

3. **Q: A team refuses to use the platform's pipeline template. What do you do?**
   A: Understand why (missing feature, slower, different language, bad experience). If it's a valid gap, add it or provide an extension point. Make sure mandatory guardrails (security scans, signing) apply to their custom pipeline too, for example via required checks or admission policies, so safety doesn't depend on the template. Adoption should be earned; compliance is enforced at the guardrail level.
