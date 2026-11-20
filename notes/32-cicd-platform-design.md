# 32 · CI/CD Platform Design

> **Level:** ★★★ advanced
>
> **You will learn:** how to design CI/CD **as a platform** rather than per-repo scripts: the **control plane vs execution plane** split, **runner/agent architectures** (static, autoscaled, ephemeral, Kubernetes-based), **pipeline templates** (reusable workflows, shared libraries), **caching** layers, **artifact flow** and registries, **monorepo vs polyrepo** effects, **multi-tenancy and isolation**, **secrets and cloud auth via OIDC**, **supply-chain security** (SBOM, signing, provenance, SLSA), measuring **pipeline speed and reliability**, and the platform's own **HA/DR**.
>
> **You need:** topics 12, 20, 21, 28. Your Jenkins and GitHub Actions notes are the hands-on side: [Jenkins 11 · architecture](../../Jenkins/notes/11-jenkins-architecture.md), [Jenkins 50 · agents](../../Jenkins/notes/50-agents-explained.md), [GHA 07 · runners](../../GitHub-Actions-Basics-To-Production/notes/07-runners.md), [GHA 22 · reusable workflows](../../GitHub-Actions-Basics-To-Production/notes/22-reusable-workflows.md).
>
> **Part 6 · Delivery** · Previous: [31 · Capacity planning](31-capacity-planning-and-performance-testing.md) · Next: [33 · Releases, promotion, GitOps](33-releases-promotion-gitops.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A CI/CD platform turns a Git event into a tested, signed, deployable artifact, and then into a running release, **for every team**, quickly and safely.

```
         CONTROL PLANE (schedules, stores state)            EXECUTION PLANE (does the work)
Git push ─▶ [CI server: GitHub Actions / Jenkins / GitLab] ──queue──▶ [Runners/agents: ephemeral pods/VMs]
                    │ pipeline templates, policies, secrets            │ build, test, scan
                    │                                                  ▼
                    │                                   [Caches]  [Artifact registry: images, packages]
                    ▼                                                  │ signed + SBOM
            [CD: GitOps (Argo CD/Flux) or pipeline deploy] ◀──────────┘
                    ▼
               [Kubernetes / cloud]
```

**Picture it.** An airport. The **control tower** (CI server) schedules flights; **aircraft** (runners) do the flying; standard **flight plans** (templates) make every flight follow the same safety checklist; **security screening** (scans, signing) happens before boarding; **cargo storage** (registry) holds what's shipped.

**The one thing to know.** **Ephemeral, isolated runners + shared templates + good caching** is the modern default: every job starts clean (no "works on runner 3" drift, no leaked secrets between jobs), templates give every team security and speed for free, and caches recover the speed lost by starting clean.

**Try it now:** for one of your pipelines, write down: queue time, total duration, and where the time goes (checkout, dependency install, build, test, image push). That's the starting point of every CI optimization.

**You've got it if you can:** draw the control/execution split, choose a runner model, design caching, and explain OIDC cloud auth and artifact signing.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Component | Job | Examples |
|---|---|---|
| **Source control + events** | Triggers (push, PR, tag, schedule) | GitHub, GitLab, Bitbucket |
| **CI orchestrator (control plane)** | Parses pipeline definitions, queues jobs, tracks state, shows logs, stores secrets | GitHub Actions, Jenkins controller, GitLab CI, Buildkite, CircleCI, Tekton |
| **Runners / agents (execution plane)** | Run job steps | GHA runners (hosted / self-hosted / Actions Runner Controller), Jenkins agents (K8s plugin), GitLab runners |
| **Pipeline templates** | Standard, reusable stages | GHA reusable workflows & composite actions, Jenkins shared libraries, GitLab includes |
| **Caches** | Dependencies, build outputs, image layers | actions/cache, S3, BuildKit cache, Bazel/Gradle remote cache |
| **Artifact registry** | Store versioned outputs | ECR/GAR/ACR, Artifactory, Nexus, GHCR, CodeArtifact |
| **CD** | Deploy to environments | Argo CD, Flux, Spinnaker, pipeline deploy jobs (topic 33) |
| **Policy & security** | Gates and supply-chain controls | Branch protection, required checks, OPA/Conftest, Sigstore/cosign, SBOM tools |

**CI** = Continuous Integration: every change is built and tested automatically, often. **CD** = Continuous Delivery (always releasable, deploy with a button) or Continuous Deployment (every passing change deploys automatically). ([Jenkins 01](../../Jenkins/notes/01-what-is-ci-cd.md), [GHA 01](../../GitHub-Actions-Basics-To-Production/notes/01-what-is-ci-cd.md).)

---

## Why it exists

At 5 engineers, a Jenkinsfile per repo is fine. At 500 engineers and 300 repos:
- Every team reinvents pipelines → inconsistent security, slow builds, duplicated effort.
- Shared static build servers become pets: dirty workspaces, disk full, credentials lying around, tool version drift.
- Queue times grow at peak hours; builds are slow → developers wait → productivity drops (DORA metrics, topic 39).
- Supply-chain attacks target CI (it holds deploy credentials and builds what runs in prod).

A platform approach fixes these once, for everyone.

---

## Analogy

The airport (Core). Extras:
- **Ephemeral runners** = a freshly cleaned aircraft for every flight; no leftover luggage (files, credentials) from the previous passengers.
- **Caching** = pre-loaded catering trucks waiting at the gate instead of cooking from scratch per flight.
- **OIDC** = instead of every pilot carrying a master key, the tower issues a one-time boarding pass valid for this flight only.

---

## How it works inside

### End-to-end flow of one change

```mermaid
sequenceDiagram
    participant Dev
    participant Git as GitHub
    participant CI as CI control plane
    participant R as Ephemeral runner (pod)
    participant C as Cache (S3/BuildKit)
    participant Reg as Registry (ECR)
    participant GO as GitOps repo + Argo CD
    Dev->>Git: 1. push / open PR
    Git->>CI: 2. webhook event
    CI->>CI: 3. resolve template, queue job (labels: linux-x64-large)
    CI->>R: 4. autoscaler starts runner pod, job assigned
    R->>C: 5. restore dependency + layer cache
    R->>R: 6. build, unit tests, lint, SAST, IaC scan
    R->>Reg: 7. push image (tag = git SHA), SBOM, signature, provenance
    R->>C: 8. save cache
    R-->>CI: 9. results, logs; runner destroyed
    CI->>Git: 10. status checks on PR
    Note over Dev,Git: merge to main (branch protection: checks + review)
    CI->>GO: 11. update image tag for dev env (PR or commit)
    GO->>GO: 12. Argo CD syncs dev; promotion to stage/prod later (topic 33)
```

### Runner models

| Model | How | Pros | Cons | Use |
|---|---|---|---|---|
| **Static long-lived VMs** | Fixed agents always on | Simple, warm caches on disk | Drift, dirty state, cross-job secret leakage, pay when idle, capacity fixed | Legacy, special hardware |
| **Autoscaled VMs** | ASG/VMSS adds VMs per queue depth | Isolation per VM (if one job per VM), cost scales | Slower start (1–3 min) | Docker-heavy builds needing full VMs |
| **Ephemeral K8s pods** ✅ | One pod per job (Actions Runner Controller, Jenkins Kubernetes plugin, GitLab K8s executor) | Fast start, clean every time, bin-packing, autoscale with cluster | Docker builds need care (rootless BuildKit/Kaniko, not privileged DinD), noisy neighbours | Most workloads |
| **Vendor-hosted** | GitHub-hosted/GitLab SaaS runners | Zero ops | Cost at scale, no private network access unless configured, limits | Small/medium orgs, OSS |
| **MicroVM-isolated** (Firecracker-based) | Each job in a lightweight VM | VM-level isolation with fast start | Newer tooling | Untrusted code, multi-tenant |

**Isolation tiers:** public/untrusted PRs (forks) must never run on runners that have prod credentials or network access to internal systems ([GHA 37 · security hardening](../../GitHub-Actions-Basics-To-Production/notes/37-security-hardening.md), [GHA 38 · self-hosted runners](../../GitHub-Actions-Basics-To-Production/notes/38-self-hosted-runners-deep-dive.md)). Separate runner pools (labels/groups) per trust level: `untrusted`, `build`, `deploy-nonprod`, `deploy-prod`.

### Pipeline templates (golden pipelines)

```
team repo: .github/workflows/ci.yml
   uses: platform-org/pipelines/.github/workflows/service-ci.yml@v3
   with: { language: java, image-name: orders }
platform template v3 (versioned, owned by platform team):
   checkout → cache restore → build → unit test → SAST → dependency scan → image build (BuildKit)
   → image scan → SBOM + sign + provenance → push → report
```
- **Versioned** (`@v3`) so the platform can evolve without breaking teams; teams upgrade on their schedule, with deprecation windows.
- **Composable:** teams can add steps but not remove mandatory security stages (enforced by required workflows / org rulesets / policy checks).
- Jenkins equivalent: shared libraries ([Jenkins 55–58](../../Jenkins/notes/55-shared-libraries-concepts.md)).

### Caching layers (speed)

| Layer | What | Where |
|---|---|---|
| Dependency cache | Maven `.m2`, npm, pip, Go modules | `actions/cache` (keyed by lockfile hash), S3 |
| Package proxy | Mirrors public registries; protects from outages and rate limits (Docker Hub, npm) | Artifactory/Nexus/CodeArtifact, ECR pull-through cache |
| Docker layer cache | Reuse unchanged image layers | BuildKit `--cache-to/--cache-from type=registry` ([Docker 16](../../Docker/notes/16-buildkit-and-build-cache.md)) |
| Build/test result cache | Skip unchanged targets | Bazel/Gradle/Nx/Turborepo remote cache |
| Runner image | Pre-installed toolchains | Custom runner images (rebuilt weekly, scanned) |
| Git | Large repo clones | Shallow/partial clone, reference repos |

Cache keys must include everything that changes the output (lockfile hash, OS, tool version) or you'll get **cache poisoning** / wrong builds. Never cache secrets.

### Monorepo vs polyrepo

| | Monorepo | Polyrepo |
|---|---|---|
| CI challenge | Build only what changed (affected-target detection, path filters, Bazel/Nx) | Many pipelines to keep consistent (templates!) |
| Pros | Atomic cross-project changes, one toolchain | Independent, simple per-repo CI |
| Cons | Huge CI load without smart change detection; big clones | Cross-repo changes are hard; version drift |

### Secrets and cloud auth

- **OIDC (OpenID Connect) federation:** the CI platform issues a short-lived signed token describing the job (repo, branch, environment); the cloud (AWS IAM role trust policy) exchanges it for temporary credentials **only** if the claims match (e.g. `repo:org/orders:environment:prod`). **No long-lived cloud keys in CI** ([GHA 36](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md)).
- **Secrets manager integration** for other secrets (Vault, AWS Secrets Manager) fetched at runtime (topic 37).
- **Environments with approvals and protection rules** for prod ([GHA 28](../../GitHub-Actions-Basics-To-Production/notes/28-environments.md)).
- Mask secrets in logs; forbid `pull_request_target` + checkout of untrusted code patterns.

### Supply-chain security (SLSA)

**SLSA** (Supply-chain Levels for Software Artifacts) = a framework of increasing guarantees that an artifact was built from the claimed source by a trusted, tamper-resistant build process.

```
Source (protected branches, reviews, signed commits)
  → Build on ephemeral isolated runner from a pinned template
  → Generate SBOM (Software Bill of Materials: list of every component and version) (Syft)
  → Sign the image (cosign / Sigstore, keyless via OIDC identity)
  → Attach provenance attestation (who/what/where built it, from which commit)
  → Registry: immutable tags, scan on push
  → Cluster admission policy (Kyverno / Gatekeeper / Sigstore policy-controller): only run images
     signed by our CI identity, from our registry, without critical vulns
```
([GHA 40 · attestations](../../GitHub-Actions-Basics-To-Production/notes/40-supply-chain-attestations.md), [Docker 43 · image supply chain](../../Docker/notes/43-image-supply-chain.md), [K8s 63](../../CKA-Certification-Course-2025/notes/63-supply-chain-and-secrets-in-production.md).) Also pin third-party actions/plugins by commit SHA.

### Measuring the platform

| Metric | Target idea |
|---|---|
| **Queue time** (job queued → started) | p95 < 1–2 min (SLO) |
| **Pipeline duration** (PR CI) | p50 < 10 min |
| **Flaky test rate** | Tracked per test; quarantine flaky tests |
| **Platform-caused failure rate** | < 0.5% of jobs (runner lost, cache/registry errors) |
| **Cost per build / per engineer** | Trend down |
| **DORA metrics** (topic 39) | Deployment frequency, lead time, change failure rate, time to restore |

### HA and DR of the platform itself

- CI is on the **recovery path** (rollback, hotfix). If CI is down during an incident, you can't ship the fix → treat it as tier-1 internally.
- **Jenkins controller** is a stateful pet: back up `JENKINS_HOME`, configuration as code (JCasC), fast rebuild, or move to an HA-friendly product ([Jenkins 67 · JCasC](../../Jenkins/notes/67-configuration-as-code.md), [Jenkins 68 · backup](../../Jenkins/notes/68-backup-restore-upgrades.md)).
- SaaS CI (GitHub Actions) outages happen → keep a documented **break-glass deploy path** (e.g. Argo CD manual sync of a known image, or a minimal runbook with scoped credentials).
- Registries: replicate images across regions; pull-through caches mean a Docker Hub outage doesn't stop builds.

---

## Options and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| CI product | GitHub Actions / GitLab CI / Jenkins / Buildkite | Use what integrates with your SCM; Jenkins for heavy customization/on-prem, at a higher ops cost |
| Runners | Hosted vs self-hosted ephemeral K8s | Hosted for simplicity; self-hosted for private network, special hardware, cost at scale |
| Container builds in K8s | Rootless BuildKit / Kaniko / remote BuildKit service / privileged DinD (⚠️) | Avoid privileged DinD on shared clusters |
| Deploy style | Push from CI vs GitOps pull | GitOps for K8s (no cluster creds in CI, drift correction) — topic 33 |
| Templates | Central mandatory vs optional library | Mandatory security stages, optional rest |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Long queue times at peak** | Devs waiting 20 min to start | Queue time SLI | Autoscaling runners on queue depth; warm pool; capacity plan (Little's Law) |
| **Runner disk full / dirty state** | Random failures on some runners | Failure by runner | Ephemeral runners |
| **Docker Hub rate limit / registry outage** | Builds fail pulling base images | `toomanyrequests` errors | Pull-through cache, mirror base images |
| **Cache poisoning / stale cache** | Wrong builds, flaky behaviour | Diffs between cached/uncached builds | Correct keys, scoped caches per branch, no writes from untrusted PRs |
| **Credential leak from CI** | Attacker uses CI secrets | Cloud audit logs, secret scanners | OIDC short-lived creds, least privilege per pipeline, isolated runner pools |
| **Compromised third-party action/plugin** | Malicious code in build | Supply-chain alerts | Pin by SHA, allow-list actions, review updates |
| **Flaky tests** | Retries, distrust in CI | Flake rate | Quarantine, owners, fix SLAs |
| **CI down during incident** | Can't roll back | — | Break-glass path, GitOps rollback independent of CI |
| **Template change breaks 300 repos** | Mass failures | Failure spike after template release | Versioned templates, canary template rollout to a few repos first |

**3 AM:** "Production hotfix needed, but CI runners won't start." Check the runner autoscaler (ARC controller logs, Karpenter capacity, quota), registry/cache reachability. Use the break-glass path: deploy the previously built, signed image via Argo CD rollback rather than building new.

---

## Where you'll use it

- **Case study 47:** CI/CD platform for 500 engineers builds on this.
- **Jenkins project 4** ([Jenkins 62](../../Jenkins/notes/62-project-4-part-1-scan-build-push.md)) and **GHA capstone** ([GHA 42](../../GitHub-Actions-Basics-To-Production/notes/42-capstone-00-overview.md)) are single-pipeline versions of this.
- **Terraform and Ansible in CI** ([Terraform 57](../../Terraform/notes/57-github-actions.md), [Ansible 53](../../Ansible/notes/53-ansible-in-github-actions.md)).

---

## Design exercise

**Prompt:** 120 repos, Jenkins with 8 static VM agents, average queue time 15 min at 11:00–16:00, builds use long-lived AWS keys stored in Jenkins, Docker builds use the host Docker socket. Propose a target architecture and migration plan.

<details>
<summary><b>Model answer</b></summary>

**Target**
- Keep Jenkins controller short-term (migration cost), but move agents to **ephemeral pods** on a dedicated EKS "ci" cluster (Jenkins Kubernetes plugin), autoscaled by Karpenter (Spot for build pools, on-demand for deploy pools). Long-term option: GitHub Actions with Actions Runner Controller if the SCM is GitHub.
- **Image builds:** rootless BuildKit (or a remote BuildKit service) with registry cache; no Docker socket mounting.
- **Auth:** replace long-lived AWS keys with **IRSA / EKS Pod Identity** for the runner pods (role per pipeline class) or OIDC from the CI system; revoke old keys.
- **Templates:** Jenkins shared library v1 with standard stages (build, test, SAST, scan, SBOM, sign, push); repos migrate to `standardPipeline(language: 'java')`.
- **Caches:** S3-backed dependency cache, ECR pull-through cache, BuildKit registry cache.
- **CD:** builds update a GitOps repo; Argo CD deploys (Jenkins no longer needs cluster credentials) (topic 33).
- **Controller resilience:** JCasC, job DSL, nightly `JENKINS_HOME` backups, restore drill.

**Sizing:** measure arrival rate at peak (e.g. 10 builds/min × 8 min avg = 80 concurrent) → pod pool scaling to ~100, nodes sized to bin-pack.

**Migration:** pilot with 5 repos → fix template gaps → migrate in waves of 20 → decommission static agents → rotate/remove AWS keys last (after all pipelines use roles).

**Metrics:** queue time p95 < 2 min, duration p50, platform failure rate, cost per build.
</details>

---

## Quick recap

- Control plane (orchestrator) vs execution plane (runners); ephemeral isolated runners by default.
- Versioned pipeline templates give every team security and speed; enforce mandatory stages.
- Caching (dependencies, package proxy, layers, build outputs) recovers speed lost to clean runners.
- OIDC short-lived cloud creds, isolated runner pools by trust level, pinned third-party actions.
- Supply chain: SBOM, signing, provenance, admission policies; measure queue time, duration, flakiness, cost; plan CI HA/break-glass.

### Interview questions

1. **Q: Why ephemeral runners instead of a fleet of long-lived build servers?**
   A: Each job starts from a known clean state, so there's no drift, leftover files, or secrets leaking between jobs, and a compromised job can't persist. They autoscale with demand so you don't pay for idle capacity. The cost is slower cold starts and lost local caches, which you fix with remote caches, package proxies and pre-built runner images.

2. **Q: How should CI authenticate to AWS?**
   A: With OIDC federation: the CI issues a short-lived token with claims (repo, branch, environment), and an IAM role trust policy allows only specific claims to assume a least-privilege role for minutes. No static keys to leak or rotate. In-cluster runners can use IRSA/Pod Identity similarly.

3. **Q: How do you prevent a malicious PR from stealing deploy credentials?**
   A: Run untrusted PR jobs on isolated runner pools with no secrets and no internal network access; never use privileged triggers with checked-out untrusted code; scope OIDC roles to protected branches/environments with approvals; separate build and deploy stages so deploy credentials exist only in protected environments; pin third-party actions by SHA.
