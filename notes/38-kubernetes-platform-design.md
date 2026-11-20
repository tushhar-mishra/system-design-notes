# 38 · Kubernetes Platform Design

> **Level:** ★★★ advanced
>
> **You will learn:** how to design Kubernetes **as a platform for many teams**: **one cluster vs many** (by environment, region, tenant, criticality), **managed vs self-managed** control planes, **multi-tenancy** models (namespaces, quotas, RBAC, network policies, policy engines, virtual clusters), **node pool** design (system/general/compute/GPU/Spot), **networking** (CNI, IP planning, ingress/Gateway, egress), the **autoscaling layers**, **platform add-ons** (the "day-1 stack"), **upgrade strategy** (in-place vs blue/green clusters), **cluster limits**, and **cost** levers.
>
> **You need:** topics 08–12, 23, 26, 33, 36–37. This topic connects to most of your [Kubernetes notes](../../CKA-Certification-Course-2025/INDEX.md).
>
> **Part 7 · Infrastructure and platform** · Previous: [37 · Config and secrets](37-config-and-secrets-at-scale.md) · Next: [39 · Platform engineering](39-platform-engineering.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **Kubernetes platform** = clusters + standard add-ons + guardrails + self-service, run by a platform team so that app teams just ship workloads.

```
Fleet: prod-ap-south-1, prod-ap-southeast-1, staging, dev, ci, (tooling/mgmt)
Each cluster:
  Control plane (managed: EKS/GKE/AKS)
  Node pools: system | general (on-demand) | spot-workers | gpu | (per-tenant if needed)
  Add-ons: CNI, CoreDNS, ingress/Gateway, cert-manager, ExternalDNS, External Secrets, Karpenter,
           metrics/logging/tracing agents, policy (Kyverno/Gatekeeper), Argo CD, backup (Velero)
  Tenancy: namespace per team/app + RBAC + ResourceQuota + LimitRange + NetworkPolicy + Pod Security
Everything defined in Git (Terraform for clusters, GitOps for add-ons and apps)
```

**Picture it.** An office building run by a facilities team. Tenants (teams) get their own floors (namespaces) with key cards (RBAC), a power budget (quotas), fire walls between floors (network policies) and building rules (policies). Facilities run the lifts, power and security (add-ons). Very sensitive tenants get their own building (separate cluster).

**The one thing to know.** **Cluster boundaries are the strongest isolation and the smallest blast radius; namespaces are cheaper but softer.** Most companies land on: separate clusters per **environment** and **region** (and for very sensitive or noisy tenants), namespaces per **team/app** inside them.

**Try it now:** `kubectl get ns`, `kubectl get resourcequota -A`, `kubectl get networkpolicy -A`. No quotas and no network policies = no real multi-tenancy.

**You've got it if you can:** argue one-vs-many clusters, design namespace tenancy, choose node pools, and plan a zero-downtime cluster upgrade.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **Fleet** | All your clusters, managed consistently |
| **Tenant** | A team/app/customer sharing a cluster |
| **Soft multi-tenancy** | Trusted tenants (teams of one company) separated by namespaces + policies |
| **Hard multi-tenancy** | Untrusted tenants (external customers' code): needs cluster or VM-level isolation (separate clusters, sandboxed runtimes like gVisor/Kata, virtual clusters with care) |
| **Node pool / node group** | A set of nodes with the same instance type/labels/taints |
| **Platform add-ons** | Cluster-wide components every workload relies on |
| **Golden path** | The supported, documented way to run a service on the platform (topic 39) |

Your notes cover the building blocks: control plane ([K8s 07](../../CKA-Certification-Course-2025/notes/07-control-plane-components.md)), HA and managed K8s ([K8s 78](../../CKA-Certification-Course-2025/notes/78-ha-control-plane-and-managed-kubernetes.md)), namespaces ([K8s 26](../../CKA-Certification-Course-2025/notes/26-namespaces.md)), quotas ([K8s 41](../../CKA-Certification-Course-2025/notes/41-requests-limits-qos-quotas.md)), RBAC ([K8s 57](../../CKA-Certification-Course-2025/notes/57-rbac.md)), admission ([K8s 58](../../CKA-Certification-Course-2025/notes/58-admission-control.md)), Pod Security ([K8s 60](../../CKA-Certification-Course-2025/notes/60-pod-security-seccomp-apparmor.md)), network policies ([K8s 61](../../CKA-Certification-Course-2025/notes/61-network-policies.md)), taints/affinity ([K8s 43–46](../../CKA-Certification-Course-2025/notes/45-taints-vs-affinity.md)), upgrades ([K8s 76](../../CKA-Certification-Course-2025/notes/76-versions-skew-and-upgrades.md)).

---

## Why it exists

Without a platform design, each team creates its own cluster (dozens of snowflakes, each with different add-ons and versions, each needing upgrades every few months) or everyone shares one unguarded cluster (one team's runaway job evicts another's pods, a misconfigured NetworkPolicy-free namespace can reach the payments DB). Kubernetes releases a minor version roughly every 4 months, and managed providers support each for a limited time, so upgrades are a permanent workload that must be designed for.

---

## Analogy

Office building (Core). **Upgrading a cluster in place** = renovating the building's lifts while people use them; **blue/green clusters** = building a new building next door, moving tenants floor by floor, then demolishing the old one.

---

## How it works inside

### One cluster vs many

| Split by | Why | Cost |
|---|---|---|
| **Environment** (dev/staging/prod) ✅ almost always | Prod isolation; test upgrades in non-prod first | More clusters to run |
| **Region** ✅ for multi-region | Failure isolation, latency, data residency (topic 42) | Fleet management |
| **Criticality / compliance** (PCI workloads) | Smaller audit scope, stricter controls | |
| **Tenant trust** (external/untrusted code) | Hard isolation | |
| **Team** (cluster per team) | Autonomy, blast radius | Many clusters, low utilization, upgrade burden |
| **Scale limits** | Very large orgs hit API server/etcd limits | |

| | Few large shared clusters | Many small clusters |
|---|---|---|
| Utilization / cost | Better bin-packing, shared add-ons | Overhead per cluster (control plane fee, add-ons, idle capacity) |
| Blast radius | Larger (bad add-on upgrade hits everyone) | Smaller |
| Isolation | Soft (namespaces) | Strong |
| Operations | Fewer upgrades, but riskier | More upgrades, needs fleet automation (GitOps, cluster templates) |
| Noisy neighbours | Need quotas/priorities | Rare |

Typical answer: **per environment × per region**, plus dedicated clusters for special compliance or heavy/untrusted workloads (CI runners often get their own cluster, topic 47).

### Tenancy inside a cluster (soft multi-tenancy checklist)

```
Namespace per team-app-env (e.g. payments-api)
├─ RBAC: team group → edit role in their namespaces only (via SSO groups)
├─ ResourceQuota (CPU/memory requests & limits, PVCs, LoadBalancers, object counts)
├─ LimitRange (default requests/limits so pods without them are still scheduled sensibly)
├─ NetworkPolicy: default deny ingress (and ideally egress), allow from ingress gateway + named callers
├─ Pod Security Admission: "restricted" (no root, no privileged, no hostPath)
├─ Policy engine (Kyverno/Gatekeeper): images only from our registry & signed, required labels (team, cost-center),
│   no :latest, probes required, resource requests required
├─ PriorityClasses: critical > standard > batch (preemption order)
└─ ServiceAccount per app with workload identity (cloud IAM per app, topic 37)
```
Automate namespace creation ("namespace vending") from a request in Git so every namespace gets the full set.

### Node pools

| Pool | Purpose | Notes |
|---|---|---|
| **System** | CoreDNS, CNI, ingress, monitoring agents, Karpenter | On-demand, tainted `CriticalAddonsOnly`, small but HA across AZs |
| **General** | Most stateless services | On-demand + some Spot; Karpenter chooses instance types |
| **Spot / batch** | Workers, CI jobs, batch | Taint so only tolerant workloads land; diversify instance types; PDBs |
| **Memory / compute optimized** | Caches, JVM-heavy, data processing | |
| **GPU** | ML inference/training | Taints, device plugins, expensive: autoscale to zero |
| **Stateful** (if self-hosting stateful) | Kafka, DBs via operators | Local NVMe or gp3; careful disruption settings |

Use **topology spread constraints** so replicas spread across AZs and nodes ([K8s 46](../../CKA-Certification-Course-2025/notes/46-pod-affinity-and-topology-spread.md)); **PodDisruptionBudgets** so drains/consolidation never take too many replicas down.

### Networking decisions

- **CNI:** AWS VPC CNI (pods get VPC IPs: simple routing, IP-hungry, topic 08) vs overlay/eBPF CNIs like Cilium (IP-efficient, network policies, observability).
- **IP planning:** big subnets or prefix delegation; secondary CIDR ranges for pods.
- **Ingress/Gateway:** one shared ingress/Gateway per cluster (with per-team routes) vs per-team gateways (isolation, more cost). Gateway API's roles model (infra owner vs route owner) fits platforms ([K8s 72](../../CKA-Certification-Course-2025/notes/72-gateway-api-concepts.md)).
- **Egress:** NAT per AZ, VPC endpoints, egress gateway/firewall for allow-listing external domains (topic 40).
- **Service mesh:** only if needed (topic 10).
- **DNS:** CoreDNS scaled + NodeLocal DNSCache (topic 07).

### Autoscaling layers

```
Pods:   HPA (requests/s, CPU) / KEDA (queues) ; VPA in recommendation mode to tune requests
Nodes:  Karpenter (or Cluster Autoscaler) ; consolidation for cost ; disruption budgets for safety
Capacity safety: overprovisioning placeholder pods ; quotas so one tenant can't take all nodes
```
(topic 12.)

### The platform add-on stack (day-1)

| Area | Typical choice |
|---|---|
| GitOps | Argo CD / Flux (topic 33) |
| Ingress/TLS/DNS | Gateway/ingress controller, cert-manager, ExternalDNS |
| Secrets | External Secrets Operator / CSI driver (topic 37) |
| Policy | Kyverno or OPA Gatekeeper; Pod Security Admission |
| Observability | Prometheus agent/OTel Collector, log agent (Fluent Bit), kube-state-metrics (topic 27) |
| Autoscaling | Karpenter, KEDA, metrics-server |
| Backup | Velero (K8s objects + volume snapshots), etcd snapshots if self-managed |
| Security | Image scanning, runtime detection (Falco), signed image admission |
| Cost | Kubecost/OpenCost with labels per team (topic 43) |

All add-ons are installed by GitOps from a **versioned platform bundle**, the same across clusters (with per-cluster values).

### Cluster upgrades

```
In-place (common with managed K8s):
 1. Read release notes; check deprecated APIs (pluto/kubent) and add-on compatibility matrix
 2. Upgrade dev → staging (soak a week) → prod (one region first)
 3. Control plane upgrade (managed: one click/Terraform; one minor version at a time)
 4. Add-ons upgraded to compatible versions
 5. Node pools: new nodes on new version (surge), cordon+drain old ones respecting PDBs
    (Karpenter drift replacement / managed node group rolling update)
 Rollback: control plane can't be downgraded → test well before prod

Blue/green clusters:
 1. Build new cluster (new version) from the same Terraform + GitOps bundle
 2. Deploy all apps there via GitOps (apps must be cluster-agnostic)
 3. Shift traffic gradually (DNS weights / global LB) per service
 4. Decommission old cluster
 + full rollback by shifting back; also tests DR rebuild capability
 − 2× cost during migration; stateful workloads and in-cluster state are hard to move
```

### Limits to know

Upstream guidance is roughly: ≤ 5,000 nodes, ≤ 150,000 pods, ≤ 110 pods per node (configurable; EKS limits by ENI/IPs unless prefix delegation), ≤ 300,000 containers. Practical limits come earlier: etcd size (default quota 2 GB, max recommended 8 GB), API server load from controllers listing/watching everything, large ConfigMaps/Secrets/CRDs. Many orgs choose more, smaller clusters long before these limits.

---

## Options and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| Control plane | Managed (EKS/GKE/AKS) vs self-managed (kubeadm) | Managed for almost everyone; self-managed only for on-prem/edge or special needs ([K8s 75](../../CKA-Certification-Course-2025/notes/75-kubeadm-cluster-build.md)) |
| Cluster split | Shared vs per team | Shared per env/region + namespace tenancy; dedicated for compliance/untrusted/noisy |
| Upgrades | In-place vs blue/green | In-place with good automation; blue/green for major platform changes or when stateless |
| Tenancy tools | Namespaces + policies / Capsule / vcluster (virtual clusters) | vcluster gives tenants their own API server (CRDs, admin) on shared nodes; isolation is still shared-node |
| Nodes | Managed node groups vs Karpenter | Karpenter for efficiency and Spot; managed groups for simplicity/predictability |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Noisy neighbour** | One tenant's pods evict or starve others | Node pressure evictions, throttling | Quotas, LimitRanges, PriorityClasses, dedicated pools |
| **Platform add-on outage** (CoreDNS, ingress, CNI) | All tenants affected | Add-on SLOs, synthetic checks | HA add-ons on system pool, PDBs, staged add-on upgrades |
| **Deprecated API removal at upgrade** | Deploys fail after upgrade | Pre-upgrade scans | pluto/kubent in CI, policy blocking deprecated APIs |
| **Drain blocked by PDB** | Upgrade stuck | Node stuck cordoned | PDBs that allow ≥1 disruption; fix single-replica apps |
| **IP exhaustion** | Pods stuck ContainerCreating | Subnet free IPs | Prefix delegation, secondary CIDRs |
| **Admission webhook down** | All pod creations fail (fail-closed webhook) | API errors mentioning webhook | HA webhooks, `failurePolicy` thought through, exclude kube-system |
| **Control plane overload** | `kubectl` slow, controllers lag | API server latency/429s | Find abusive clients (API Priority and Fairness metrics), reduce watch/list load |
| **Cluster-wide change breaks everyone** | Fleet-wide incident | — | Progressive rollout of platform bundle: dev → staging → one prod cluster → rest |
| **Snowflake clusters** | Inconsistent behaviour, hard upgrades | Diff between clusters | Everything from Terraform modules + GitOps bundle |

**3 AM:** "New pods can't start in prod anywhere." Check admission webhooks (a policy engine or mesh injector down with `failurePolicy: Fail`), CNI (IPs), and node capacity (Karpenter). Mitigation for webhook: scale it up or temporarily switch its failure policy per runbook.

---

## Where you'll use it

- **Case study 51** (multi-region Kubernetes) and **47** (CI on K8s).
- **Platform engineering** (topic 39): the K8s platform is usually its core product.
- **Interviews:** "How would you set up Kubernetes for 30 teams?" is a common senior question.
- **Production readiness** checklist ([K8s 85](../../CKA-Certification-Course-2025/notes/85-production-readiness-checklist.md)).

---

## Design exercise

**Prompt:** 30 product teams (~250 services), one region today (second region next year), PCI scope for the payments domain (6 services), heavy CI usage, some ML inference with GPUs. Design the cluster topology and tenancy.

<details>
<summary><b>Model answer</b></summary>

- **Clusters (EKS, Terraform module, one state per cluster):** `prod-main`, `prod-pci` (payments only: smaller audit scope, stricter network and access), `staging-main`, `staging-pci`, `dev`, `ci` (runners isolated from prod, Spot-heavy), `tooling` (Argo CD hub if centralized, observability backends). Next year: replicate `prod-main`/`prod-pci` to region 2 (topic 51).
- **Tenancy (`prod-main`):** namespace per team-service, created by namespace vending from Git with RBAC (SSO group → namespace edit), quotas, LimitRanges, default-deny NetworkPolicies, Pod Security `restricted`, Kyverno policies (signed images from ECR, required labels, probes, requests), workload identity per service.
- **Node pools (Karpenter NodePools):** system (on-demand, tainted), general (on-demand + up to 50% Spot for stateless with ≥2 replicas), batch (Spot, tainted), GPU (on-demand g-family, tainted, scale to zero, used by inference namespaces only).
- **Networking:** VPC CNI with prefix delegation, /19 pod subnets per AZ; Gateway API with a shared gateway per cluster (platform owns Gateway, teams own HTTPRoutes); egress via NAT per AZ + VPC endpoints; PCI cluster: egress firewall allow-list.
- **Add-ons:** a versioned platform bundle via Argo CD ApplicationSets; rollout order dev → staging → prod-pci → prod-main.
- **Upgrades:** quarterly; in-place via Terraform with node rotation by Karpenter drift; deprecated API checks in CI; staged a week per environment.
- **Cost:** OpenCost by namespace/team label; showback monthly (topic 43).
- Trade-off: separate PCI cluster costs more and duplicates add-ons, but shrinks compliance scope from 250 services to 6.
</details>

---

## Quick recap

- Clusters for strong isolation (per env, region, compliance, untrusted); namespaces for team tenancy inside them.
- Soft multi-tenancy = RBAC + quotas + LimitRanges + NetworkPolicies + Pod Security + policy engine + workload identity, automated by namespace vending.
- Node pools by purpose (system, general, Spot, GPU) with taints; spread across AZs; PDBs.
- Standard add-on bundle via GitOps, rolled out progressively across the fleet.
- Upgrades are continuous work: staged in-place or blue/green clusters, with deprecated API checks.

### Interview questions

1. **Q: One big cluster or many small ones?**
   A: Separate clusters where isolation or blast radius really matters: environments, regions, compliance scope, untrusted workloads. Inside them, share with namespace-based tenancy for better utilization and fewer upgrades. Per-team clusters only when teams need cluster-level autonomy and you have fleet automation, since each cluster adds cost and upgrade work.

2. **Q: How do you stop one team's workload from hurting another in a shared cluster?**
   A: Requests/limits with LimitRange defaults, ResourceQuotas per namespace, PriorityClasses, dedicated tainted node pools for heavy workloads, NetworkPolicies, Pod Security restricted, policy engine rules, per-team RBAC, and per-namespace API usage limits via API Priority and Fairness for controllers.

3. **Q: How do you upgrade a production cluster safely?**
   A: Check deprecations and add-on compatibility first, upgrade lower environments and soak, then prod one cluster/region at a time: control plane, then add-ons, then rotate node pools with surge and drains respecting PDBs. Since control planes can't be downgraded, the safety net is thorough pre-prod testing, or blue/green clusters with traffic shifting for big jumps.
