# 33 · Releases, Promotion and GitOps

> **Level:** ★★ intermediate
>
> **You will learn:** **build once, deploy many** with **immutable artifacts**, **versioning** (SemVer, Git SHA, digests), **environment promotion** flows (dev → staging → prod) and how to model environments, **push-based vs pull-based (GitOps)** deployment, how **Argo CD / Flux** work, **config vs code** per environment (Helm values, Kustomize overlays), **rollback** strategies, **release trains vs continuous deployment**, and the failure modes of GitOps (drift, sync loops, secrets).
>
> **You need:** topic 32. [Jenkins 07 · artifacts, tags and digests](../../Jenkins/notes/07-artifacts-tags-and-digests.md), [Jenkins 09 · promotion flows](../../Jenkins/notes/09-promotion-flows.md) and [K8s 68 · GitOps](../../CKA-Certification-Course-2025/notes/68-gitops.md) go hands-on.
>
> **Part 6 · Delivery** · Previous: [32 · CI/CD platform](32-cicd-platform-design.md) · Next: [34 · Deployment strategies](34-deployment-strategies.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Build an artifact **once**, give it an **immutable identity**, and **promote that same artifact** through environments by changing only configuration. With **GitOps**, the desired state of each environment lives in Git, and an agent in the cluster pulls and applies it.

```
CI: build → image orders@sha256:9f2c… (tag 1.8.0 / git sha a1b2c3)
          ↓ (same digest everywhere)
GitOps repo:  envs/dev/orders.yaml    image: …@sha256:9f2c   ← auto
              envs/staging/…          image: …@sha256:9f2c   ← PR after dev checks
              envs/prod/…             image: …@sha256:9f2c   ← PR + approval
Argo CD in each cluster: watches its env folder → applies → reports Synced/Healthy
Rollback = revert the commit in Git
```

**Picture it.** A publishing house. The book is printed **once** (artifact); the same print run goes to stores in different cities (environments) with only the price sticker changing (config). The shipping manifest (Git) says which edition each store should have; each store manager (Argo CD) checks the manifest and fixes the shelf if it doesn't match.

**The one thing to know.** **Never rebuild per environment.** If staging tested image X and prod runs a rebuild Y "from the same commit", you've shipped something untested (different base image layers, dependency versions). Promote by **digest**.

**Try it now:** `kubectl get deploy orders -o jsonpath='{.spec.template.spec.containers[0].image}'`: is it a mutable tag like `:latest` or an immutable digest?

**You've got it if you can:** design a promotion flow, compare push vs GitOps, and explain how a rollback works in each.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **Artifact** | The deployable output: container image, JAR, Helm chart, binary |
| **Immutable artifact** | Never changed after build; a new change = a new artifact |
| **Tag** | Human-friendly, **mutable** name (`1.8.0`, `latest`) |
| **Digest** | Content hash (`sha256:…`), **immutable** identity |
| **Promotion** | Moving the same artifact to the next environment after checks |
| **Release** | A version made available to users (may be decoupled from deploy via feature flags, topic 34) |
| **Push-based CD** | The pipeline runs `kubectl/helm apply` against the target |
| **GitOps (pull-based)** | Git holds desired state; an in-cluster agent pulls and reconciles continuously |
| **Drift** | Actual state differs from desired state |
| **Reconciliation** | The loop that detects and fixes drift (same idea as K8s controllers) |

**GitOps principles (OpenGitOps):** declarative desired state, versioned and immutable (in Git), pulled automatically, continuously reconciled.

---

## Why it exists

- **Reproducibility:** what you tested is exactly what you run.
- **Auditability:** every change to prod is a Git commit with author, review and timestamp.
- **Easy rollback:** revert a commit; or redeploy the previous digest.
- **Security:** with GitOps, CI doesn't hold cluster-admin credentials; the cluster pulls.
- **Drift correction:** manual `kubectl edit` changes are detected (and reverted).

---

## Analogy

Publishing house (Core). Extras:
- **Rebuilding per environment** = reprinting the book for each city: a different printer may use slightly different paper and a typo might sneak in.
- **Drift** = a store manager rearranges the shelf by hand; the next manifest check puts it back.

---

## How it works inside

### Versioning

| Scheme | Example | Good for |
|---|---|---|
| **SemVer** (MAJOR.MINOR.PATCH) | `2.4.1` | Libraries, APIs, things consumers depend on (MAJOR = breaking) |
| **Git SHA** | `a1b2c3d` | Services; ties artifact to exact source |
| **Build number / date** | `2026.11.20-142` | Ordering, human readability |
| **Digest** | `sha256:9f2c…` | The actual deployment reference (immutable) |

Common for services: tag with Git SHA (+ SemVer for releases), **deploy by digest**. Make registry tags **immutable** (ECR tag immutability) so `1.8.0` can't be overwritten.

### Promotion flow

```
PR → CI (unit, lint, scans) → merge to main
     → build image (once), sign, SBOM, push
     → auto-deploy to DEV (GitOps commit) → smoke tests, integration tests
     → promotion PR to STAGING (bot opens it; may auto-merge on green)
          → e2e tests, performance checks, security DAST
     → promotion PR to PROD (requires approval / change window / error budget check)
          → progressive rollout (canary, topic 34) → verify SLOs → done
```

**Gates** between environments: automated tests, policy checks (signed image? critical CVEs?), error budget status (topic 28), manual approval only where it adds value (regulatory, high-risk). Every manual gate slows lead time.

### Modelling environment config

- **Helm:** one chart, `values-dev.yaml`, `values-prod.yaml` ([K8s 65](../../CKA-Certification-Course-2025/notes/65-helm.md)).
- **Kustomize:** `base/` + `overlays/{dev,staging,prod}` with patches ([K8s 64](../../CKA-Certification-Course-2025/notes/64-kustomize.md)).
- **Keep differences small and explicit:** replicas, resources, URLs, feature flags. Big differences between staging and prod mean staging tests prove little.
- Secrets are **references** (External Secrets, Sealed Secrets, SOPS) never plaintext in Git (topic 37).

### Repo layouts

| Layout | Pros | Cons |
|---|---|---|
| App code + manifests in the same repo | Simple, one PR | CI commits to app repo for every promotion; noisy history; access control mixed |
| **Separate GitOps/config repo** ✅ (common) | Clear audit of what's deployed, separate permissions, promotions are config PRs | Two repos to coordinate |
| Per-environment branches | Looks intuitive | ⚠️ Merge drift between branches; prefer folders per environment on one branch |

### Push vs pull

```
PUSH:  CI runner ──kubectl/helm apply (needs cluster creds)──▶ cluster
PULL:  CI ──commit──▶ Git ◀──watch/poll── Argo CD (in cluster) ──apply──▶ cluster
```

| | Push (pipeline deploys) | Pull (GitOps) |
|---|---|---|
| Credentials | CI holds cluster/cloud deploy creds | Only the in-cluster agent has creds; CI only writes to Git |
| Drift detection | None (until next deploy) | Continuous; optional self-heal |
| Multi-cluster | CI must reach every cluster (network, creds) | Each cluster pulls; works behind firewalls |
| Visibility | Pipeline logs | Git history + Argo UI (sync status, diffs) |
| Non-K8s targets | Natural (VMs, Lambda, Terraform) | K8s-centric (Crossplane/Terraform controllers extend it) |
| Ordering/complex workflows | Easy in pipelines | Needs sync waves, hooks, Argo Rollouts/Workflows |

Many orgs use **both**: CI pushes infra changes via Terraform pipelines (topic 36), GitOps for apps on K8s.

### How Argo CD works

```
1. Application CR: source = git repo/path/revision, destination = cluster/namespace
2. Repo server renders manifests (Helm/Kustomize/plain)
3. Controller compares rendered desired state vs live state (K8s API) → OutOfSync/Synced
4. Sync (auto or manual): apply in order (sync waves, PreSync/PostSync hooks)
5. Health assessment: Deployment rolled out? Pods ready? → Healthy/Degraded/Progressing
6. Self-heal (optional): revert manual changes; Prune: delete resources removed from Git
```
At scale: **ApplicationSets** generate Applications per cluster/env from templates; one Argo CD per cluster or a central one managing many (trade-off: blast radius vs simplicity; topic 51).

### Rollback

| Method | How | Notes |
|---|---|---|
| **Git revert** (GitOps) ✅ | Revert the promotion commit; Argo syncs previous digest | Auditable; seconds to minutes |
| Argo CD rollback to previous sync | UI/CLI | Fast, but then Git ≠ cluster (disable auto-sync or revert Git too) |
| `kubectl rollout undo` / `helm rollback` | Push-based | Fast; Git drifts if GitOps |
| Progressive delivery abort | Argo Rollouts/Flagger automatically abort a bad canary (topic 34) | Fastest; no human |
| Feature flag off | Turn off the new code path (topic 34, 57) | Instant; code stays deployed |

⚠️ Rollback of code is easy; rollback of **data** isn't. DB migrations must be backward compatible so old code can run against the new schema (topic 35).

### Release cadence

- **Continuous deployment:** every merge goes to prod automatically if checks pass (high-maturity teams; small changes, low risk per change).
- **Continuous delivery:** always deployable, prod deploy on button press.
- **Release trains:** scheduled releases (weekly). Bigger batches → riskier, slower feedback; sometimes required (mobile app stores, regulated environments).
- Smaller, more frequent changes have **lower** change failure rates and faster recovery (DORA research, topic 39).

---

## Options and trade-offs

| Decision | Guidance |
|---|---|
| Deploy reference | Digest (or immutable SHA tag); never `latest` |
| Push vs GitOps for K8s apps | GitOps for security, drift and multi-cluster; push is fine for small setups |
| Auto-promotion | Auto to dev/staging; prod auto only with strong automated verification (canary analysis) |
| Config repo | Separate repo + folder per env on one branch |
| Self-heal | On for prod (prevents drift); have a documented break-glass to pause it during incidents |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Rebuilt per env** | Prod differs from what was tested | Different digests per env | Build once, promote digest |
| **Mutable tag overwritten** | Pods restarting pull different code; nodes run different versions | Digest mismatch across pods | Immutable tags, deploy by digest |
| **Manual hotfix in cluster** | Reverted by self-heal, or lingering drift | Argo OutOfSync | All changes via Git; break-glass procedure that ends with a Git commit |
| **Sync loop / perpetual OutOfSync** | Argo constantly applying | Sync count, diff | Ignore fields set by controllers (`ignoreDifferences`), fix defaults |
| **Bad commit auto-synced everywhere** | All clusters break at once | Error spike after sync | Progressive sync across envs/clusters, canary, required reviews |
| **Git provider down** | Can't deploy (can't roll back via Git) | — | Argo keeps running last state; break-glass manual sync of known revision; mirror repo |
| **Argo CD itself down** | No deploys, no drift correction (apps keep running) | Argo health | HA Argo, monitoring, it's a tier-1 platform component |
| **Secrets in Git** | Leak | Secret scanning | External Secrets/SOPS (topic 37) |

**3 AM:** "Bad release in prod." Revert the promotion commit (or abort the rollout); confirm Argo synced the previous digest; watch SLIs recover. If Git or Argo is unavailable, use the break-glass: `kubectl rollout undo` and record it, then reconcile Git afterwards.

---

## Where you'll use it

- **Case studies 47, 48, 51** (CI/CD platform, zero-downtime deploys, multi-region K8s).
- **Jenkins + GitOps** ([Jenkins 72](../../Jenkins/notes/72-jenkins-and-gitops.md)); **GHA automated promotion PRs** ([GHA 52](../../GitHub-Actions-Basics-To-Production/notes/52-capstone-10-automated-pull-request.md)).
- **Terraform:** plan → approve → apply promotion of infra changes across env folders (topic 36).
- **Ansible:** promoting the same role version through inventories ([Ansible 39](../../Ansible/notes/39-project-layout.md)).

---

## Design exercise

**Prompt:** 40 services, 3 environments (dev, staging, prod), prod in 2 clusters (two regions). Today CI runs `helm upgrade` against each cluster with a kubeconfig stored in CI, images tagged `:latest`. Design the target release and promotion system.

<details>
<summary><b>Model answer</b></summary>

- **Artifacts:** CI builds once per merge, tags with Git SHA, pushes to ECR with **immutable tags**, signs with cosign, attaches SBOM/provenance; replicated to the second region's registry.
- **Config repo:** `platform-gitops` with `apps/<service>/base` (Helm chart or Kustomize) and `envs/{dev,staging,prod-ap-south-1,prod-ap-southeast-1}/<service>/values.yaml` holding the **digest**.
- **Argo CD:** one instance per cluster (blast radius contained), ApplicationSets generating one Application per service per env. Auto-sync + self-heal + prune on.
- **Promotion:** CI commits the new digest to `envs/dev` automatically. After dev checks pass, a bot opens a PR to staging (auto-merge on green). For prod: bot opens one PR updating **prod region 1 first**; after its canary (Argo Rollouts with analysis on SLIs) succeeds and a soak period passes, a second PR updates region 2. Approvals via CODEOWNERS for prod folders.
- **Credentials:** remove kubeconfigs from CI; CI only needs write access to the GitOps repo (scoped token / GitHub App).
- **Admission:** Kyverno policy in prod clusters: only signed images from our registry, by digest.
- **Rollback:** revert PR; Argo Rollouts auto-abort on failed analysis. Break-glass documented.
- **Metrics:** lead time (merge → prod), deploy frequency, change failure rate, time to restore.
</details>

---

## Quick recap

- Build once, deploy many: immutable artifacts, deploy by digest, immutable tags.
- Promote the same artifact dev → staging → prod with automated gates; manual approval only where it adds value.
- GitOps: Git = desired state, in-cluster agent pulls and reconciles; no cluster creds in CI; drift detection; rollback = git revert.
- Keep env differences small; secrets as references.
- Code rollback is easy, data rollback isn't: backward-compatible migrations.

### Interview questions

1. **Q: Why build once and promote instead of building for each environment?**
   A: So the exact bytes tested in staging are what runs in prod. Rebuilding can pull different dependency versions or base layers, invalidating test results. Promoting by digest also makes rollbacks precise and audits simple.

2. **Q: Push-based deploys vs GitOps: what are the trade-offs?**
   A: Push is simple and works for any target, but CI needs powerful credentials for every cluster and there's no drift detection. GitOps keeps credentials in the cluster, gives an auditable Git history, continuously corrects drift and scales to many clusters, but is K8s-centric and needs care for ordering, secrets and break-glass procedures.

3. **Q: How do you roll back with GitOps, and what can't you roll back?**
   A: Revert the commit that changed the image digest/config; the agent syncs back to the previous state. Progressive delivery can abort automatically. What you can't simply roll back is data and schema changes, so migrations must be backward compatible (expand/contract) and destructive steps happen only after the new version is proven.
