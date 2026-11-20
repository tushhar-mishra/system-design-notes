# 47 · Case Study: A CI/CD Platform for 500 Engineers

> **Level:** ★★★ advanced
>
> **You will practise:** designing CI/CD as a **multi-tenant platform**: requirements from engineering teams, **runner fleet sizing** with Little's Law, **autoscaled ephemeral runners** on Kubernetes, **queueing and fairness**, **caching and artifact flow**, **security isolation and supply chain**, **GitOps CD**, the platform's own **SLOs, HA and DR**, and **cost per build**.
>
> **You need:** topics 03, 12, 20–21, 28, 31–34, 36–40, 43.
>
> **Part 9 · Case studies** · Previous: [46 · HA web app](46-case-ha-web-app.md) · Next: [48 · Zero-downtime deployments](48-case-zero-downtime-deployments.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design a CI/CD platform for a company with 500 engineers."

| Question | Assumed answer |
|---|---|
| SCM? | GitHub Enterprise Cloud |
| Repos / languages? | ~400 repos: Java, Go, Node, Python; a few Android/iOS apps; one large monorepo for the web frontend |
| Deploy targets? | EKS (most services), some Lambda, Terraform for infra |
| Current pain? | Queue times of 15–30 min at peak, 25-min pipelines, inconsistent pipelines, static AWS keys in CI |
| Requirements | p95 queue time < 2 min; PR feedback < 10 min for most repos; secure supply chain; deploy to prod many times per day; self-service onboarding |
| Special hardware? | macOS for iOS builds; a few GPU test jobs; ARM builds for Graviton |
| Compliance? | SOC 2: audit trail of who deployed what |
| Budget? | Cost visibility per team required |

---

## 2. Estimation

```
Pipelines: 500 engineers × ~6 runs/day = 3,000 runs/day; each run = ~4 jobs → 12,000 jobs/day
Working window ~10 h (two time zones) → 1,200 jobs/h avg; peak hour ×2 → 2,400 jobs/h = 40 jobs/min
Avg job duration today 8 min → concurrency at peak (Little's Law) = 40 × 8 = 320 concurrent runners
Target after caching (avg 5 min) → 200 concurrent at peak; night ~10–20

Runner sizes: 70% small (2 vCPU/4 GB), 25% medium (4/16), 5% large (16/64) → peak vCPU ≈ 200 × ~3.2 ≈ 640 vCPU
Compute/day: 12,000 jobs × 5 min = 1,000 runner-hours ≈ 3,200 vCPU-hours/day
   at ~$0.04/vCPU-hour on-demand (illustrative) ≈ $130/day; Spot ~60% cheaper

Artifacts: 3,000 images/day × 300 MB (layers mostly shared; ~50 MB new per image) ≈ 150 GB/day new data
Logs: 12,000 jobs × 2 MB = 24 GB/day
Caches: ~5–10 TB working set (dependencies + layer caches)
```

**Conclusion:** we need **elastic capacity** (20 → 320 runners), a **fast cache layer** (main lever for duration), **strong isolation** (multi-tenant with secrets), and **fair queueing** across teams.

---

## 3. High-level design

```
GitHub (repos, PRs, branch protection, org rulesets, required workflows)
   │ webhooks / Actions job queue
   ▼
GitHub Actions control plane (SaaS) ──────────────▶ job assigned to runner scale set by labels
   │
   ▼
┌──────────────────────── CI EKS cluster (dedicated account "ci-prod") ─────────────────────────┐
│ Actions Runner Controller (ARC) scale sets:                                                   │
│   linux-small | linux-medium | linux-large | linux-arm64 | untrusted-pr | deploy-nonprod | deploy-prod
│ Karpenter node pools: Spot (build pools), on-demand (deploy pools, large), ARM (Graviton)      │
│ Remote BuildKit service (rootless) with registry cache  ◀── image builds                       │
│ Package proxy (Artifactory / CodeArtifact + ECR pull-through) ◀── deps, base images            │
│ S3 cache bucket (actions/cache backend) via VPC endpoint                                       │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
macOS: managed macOS runners (GitHub-hosted larger runners or EC2 Mac dedicated hosts) for iOS
   │ outputs
   ▼
ECR (immutable tags, scan on push, replicated cross-region) + SBOM/signature/provenance (cosign, attestations)
   │
   ▼
GitOps repo ◀── promotion bot PRs ──▶ Argo CD per cluster ──▶ Argo Rollouts canaries
Terraform pipelines (separate runner pool, OIDC roles per account) for infra
Backstage portal: templates, catalog, CI dashboards
Observability: runner metrics, queue time SLI, job outcomes classified, cost per team (OpenCost + labels)
```

**Flow of a PR:**
1. Developer opens PR → GitHub queues jobs from the repo's workflow (which `uses:` the platform's reusable workflow `@v4`).
2. ARC sees pending jobs for `linux-medium` → scales the runner set → Karpenter adds nodes if needed (warm buffer keeps p95 start < 1 min).
3. Ephemeral runner pod registers (just-in-time config), runs the job: restore caches → build/test → scans → (on main) build image via remote BuildKit → push to ECR → sign + attest.
4. Runner pod is deleted after one job.
5. Status checks reported; merge allowed when required checks pass and review is approved.
6. On merge to main: promotion bot updates `envs/dev` in the GitOps repo; later stages as in topic 33.

---

## 4. Deep dives

### 4.1 Runner fleet and autoscaling
- **One job per runner pod** (ephemeral), runner image with toolchains pre-installed (rebuilt weekly, scanned).
- **Scale sets per class** with `minRunners` warm buffer during working hours (e.g. 20 small, 5 medium) via scheduled config; 0–2 at night.
- **Karpenter** consolidation + Spot diversification for build pools; interruption → job fails → automatic retry policy for infra-classified failures.
- **Quotas per team** (max concurrent runners per org/team via runner groups) to stop one team's matrix of 300 jobs starving others; **priority** for `main`-branch and deploy jobs over PR jobs.

### 4.2 Speed: caching and build design
| Lever | Expected gain |
|---|---|
| Dependency cache (S3-backed, keyed by lockfile) + package proxy | −2–4 min per job |
| BuildKit registry cache (`cache-from/to`) | Image builds 6 min → 1–2 min |
| Monorepo: affected-target detection (Nx/Bazel remote cache) | Only changed projects built |
| Test sharding (matrix) and parallel stages | Long test suites split |
| Flaky test quarantine | Fewer reruns |
| Pre-pulled runner images on nodes (DaemonSet image warmers) | Faster pod start |

### 4.3 Security and isolation
- **Trust tiers:** `untrusted-pr` pool (fork PRs, external contributors): no secrets, no internal network (NetworkPolicy egress only to proxy), separate node pool. Internal PR builds: read-only package access. `deploy-*` pools: only reachable from protected branches/environments.
- **OIDC** from GitHub to AWS: role trust conditions on `repo`, `ref`, `environment` (e.g. only `environment:prod` from `main` can assume `deploy-prod`) (topic 32).
- **No Docker socket / privileged pods:** rootless BuildKit.
- **Supply chain:** pinned actions by SHA (allow-list via org policy), required platform workflow for SBOM + signing (enforced via org rulesets/required workflows), Kyverno in prod clusters verifies signatures and provenance.
- **Secrets:** environment secrets for few remaining needs; everything else via OIDC → Secrets Manager.
- **Audit:** GitHub audit log + deployment records (who approved, which digest) → SIEM.

### 4.4 CD
GitOps (Argo CD per cluster) with promotion PRs; Argo Rollouts canaries; Terraform via plan-on-PR/apply-on-merge pipelines with policy checks (topic 36). CI never holds cluster credentials.

---

## 5. Scaling

- Concurrency from 20 to 400+ runners via ARC + Karpenter; cluster sized for ~800 vCPU peak with headroom; vCPU quotas raised.
- GitHub API rate limits: ARC uses webhooks/long-poll scale sets; bots use GitHub Apps (higher limits).
- Cache bucket and package proxy scale horizontally (S3 unlimited; proxy replicas behind LB).
- Monorepo growth: remote build cache becomes essential; consider merge queue to serialize main-branch validation.

---

## 6. Reliability and DR

**Platform SLOs** (published to engineering):
- Queue time: 99% of jobs start within 2 min.
- Platform-caused job failure rate < 0.5%.
- Deploy pipeline availability 99.9% during business hours.

**Failure walkthrough:**
| Failure | Impact | Mitigation |
|---|---|---|
| GitHub Actions outage | No CI at all | Break-glass: Argo CD can roll back to previous digest without CI; documented emergency build path on a standby runner with manual approval |
| CI cluster down | Queued jobs | Cluster defined in Terraform, rebuilt < 1 h; second CI cluster in another region for deploy pools |
| Package proxy down | Builds fail on dependency fetch | HA replicas, fallback to upstream for non-prod |
| Spot capacity drought | Slower queue | Fallback to on-demand pools (Karpenter weights) |
| Bad template release | Mass failures | Template versions; canary release to 10 volunteer repos first |
| ECR region issue | Can't push/pull | Cross-region replication; pull from replica |

---

## 7. Security (summary)
Covered in 4.3: isolation by trust tier, OIDC, no privileged builds, pinned and allow-listed actions, signing and admission verification, audit trails, secrets out of CI. Plus: dependency confusion protection via the proxy, branch protection + CODEOWNERS on workflow files (changing a pipeline requires platform review).

---

## 8. Observability

- **Metrics:** queue time per runner class, job duration by repo/step, outcome classification (code failure vs infra failure vs flaky), runner churn, cache hit ratios, Spot interruptions.
- **Dashboards per team** in Backstage: their pipeline speed, flakiness, cost.
- **DORA metrics** collected from deployment events (topic 39).
- **Alerts:** queue time SLO burn (page during business hours), platform failure rate spike, cache/proxy errors.

---

## 9. Cost

```
Compute: ~1,000 runner-hours/day; with ~70% on Spot → maybe $2–4k/month compute (illustrative)
Plus: CI cluster overhead, BuildKit service, proxy, S3 caches (~10 TB), ECR storage (lifecycle rules!), macOS hosts (expensive: dedicated hosts 24 h minimum)
Cost per build and per team via labels → showback
Biggest levers: caching (shorter jobs), Spot, right-sized runner classes, ECR lifecycle (keep last N untagged), log retention
```

---

## 10. Trade-offs

| Decision | Chosen | Alternative | Reason |
|---|---|---|---|
| Orchestrator | GitHub Actions | Jenkins / Buildkite | Tight GitHub integration, less to operate; Jenkins kept only for legacy jobs during migration |
| Runners | Self-hosted ephemeral on EKS | GitHub-hosted | Private network access, cost at scale, custom sizes; hosted kept for macOS/occasional |
| Builds | Remote rootless BuildKit | DinD privileged | Security on shared cluster |
| CD | GitOps pull | Push from CI | No cluster creds in CI, drift correction |
| Templates | Versioned reusable workflows | Per-repo pipelines | Consistency, security by default |

---

## 11. Follow-up questions

1. **Q: A team's 400-job matrix hogs all runners every morning. What do you do?**
   A: Per-team concurrency limits via runner groups/scale-set limits, lower priority for PR jobs vs main/deploy, help them shard smarter or use affected-only testing, and show them the cost/queue impact in their dashboard.

2. **Q: How do you prove to an auditor who deployed what to prod?**
   A: Every prod change is a merged PR in the GitOps repo (author, approver), tied to an image digest with signed provenance linking it to the source commit and CI run; Argo CD sync history and GitHub audit logs are retained in the SIEM.

3. **Q: How do you roll out a new version of the shared pipeline template safely?**
   A: SemVer the template, release `v5` to a canary set of repos, watch failure rate and duration, publish migration notes, let teams opt in, then bump default for new repos and open automated upgrade PRs; keep `v4` supported for a deprecation window.

4. **Q: How do you reduce PR feedback time from 25 to under 10 minutes?**
   A: Measure step breakdown first; then caching (dependencies, layers, build outputs), warm runners (queue time), test sharding and affected-only builds, moving slow integration/e2e tests to post-merge or nightly with fast smoke tests on PR, and fixing flaky tests.
