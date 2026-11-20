# 43 · Cost-Aware Design and FinOps

> **Level:** ★★ intermediate
>
> **You will learn:** the main **cloud cost drivers** (compute, storage, data transfer/egress, managed services, observability), **pricing models** (on-demand, Savings Plans/Reserved, Spot), **unit economics** (cost per request/customer/build), **tagging and allocation** (showback/chargeback), **rightsizing**, the classic **hidden costs** (NAT gateways, cross-AZ traffic, logs, idle resources), **cost vs reliability** trade-offs, Kubernetes cost allocation, and how **FinOps** works as a practice.
>
> **You need:** topics 03, 08, 12, 20, 27, 38, 41–42.
>
> **Part 8 · Security, DR and cost** · Previous: [42 · Multi-region](42-multi-region-architecture.md) · Next: [44 · Chaos engineering](44-chaos-engineering-and-game-days.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Cost is a **non-functional requirement** like latency or availability. Every design choice has a price; seniors state it.

```
Big levers (usually):
1. Compute:  right-size, autoscale, Spot for fault-tolerant work, commit (Savings Plans) for the steady baseline
2. Storage:  lifecycle to cheaper tiers, delete what nobody needs, compress
3. Network:  egress, cross-AZ, NAT gateway processing — the "invisible" bill
4. Observability: logs and high-cardinality metrics can cost as much as the app
5. Architecture: managed vs self-hosted, multi-region (≈2×), over-provisioned DR
```

**Picture it.** Running a household. Rent (baseline compute) is cheaper with a long lease (commitment). Taxis (on-demand) are flexible but pricey; a carpool (Spot) is cheap but may cancel last minute. Lights left on in empty rooms (idle resources) and the phone bill for international calls (egress) quietly add up.

**The one thing to know.** Measure **unit cost** (e.g. cost per 1,000 orders, per active customer, per build), not just the total. A growing bill can be healthy (more customers) or a leak (cost per customer rising).

**Try it now:** in AWS Cost Explorer, group last month by **Usage type**. Look for `NatGateway-Bytes`, `DataTransfer-Regional-Bytes` (cross-AZ), `CW:` logs ingestion, and idle `EBS:VolumeUsage` for unattached volumes.

**You've got it if you can:** estimate the cost of a design's main components, name 5 hidden costs, and propose savings without hurting SLOs.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **FinOps** | The practice (and cultural model) of managing cloud cost collaboratively between engineering, finance and business (FinOps Foundation: Inform → Optimize → Operate) |
| **On-demand** | Pay per second/hour, no commitment |
| **Savings Plans / Reserved Instances / Committed Use Discounts (GCP) / Reservations (Azure)** | Commit to 1–3 years of usage for big discounts (often ~30–70%) |
| **Spot / Preemptible / Spot VMs** | Spare capacity at large discounts (often 60–90%), can be reclaimed at short notice |
| **Showback / chargeback** | Showing (or billing) each team its costs |
| **Unit economics** | Cost per business unit (request, customer, transaction, build) |
| **Rightsizing** | Matching resource sizes to actual use |
| **Egress** | Data leaving a cloud region/provider to the internet or another region (charged) |

---

## Why it exists

Cloud makes spending as easy as an API call; nobody signs a purchase order for an autoscaling group. Without visibility and ownership, costs grow faster than the business. In interviews, a senior design that ignores cost (multi-region active-active for an internal tool, 30-day hot logs for 10 TB/day) is a red flag.

---

## Analogy

Household budget (Core). **Unit economics** = cost per person per month: if the family grows and the grocery bill grows proportionally, fine; if it doubles while the family is the same size, investigate.

---

## How it works inside

### Cost drivers and typical savings

| Driver | Typical waste | Levers |
|---|---|---|
| **Compute (VMs, K8s nodes)** | Over-sized instances, low utilization, idle non-prod at night | Rightsize (VPA/recommendations), autoscale, bin-pack (Karpenter consolidation), schedule non-prod off at night/weekends (~65% saving on those), **Graviton/ARM** instances (often ~20% better price-performance), Spot for stateless/batch/CI, Savings Plans for the steady base |
| **K8s requests** | Requests far above real usage → nodes full of reserved-but-unused capacity | Set requests from measured p95 usage, VPA recommendations, review "requested vs used" per namespace |
| **Databases** | Over-provisioned instances, too many replicas, provisioned IOPS unused | Rightsize, Aurora Serverless v2 for spiky loads, reserved instances, gp3 over io1 where possible |
| **Storage** | Old snapshots, unattached volumes, logs/backups never expired, everything in hot tier | Lifecycle policies (topic 20), snapshot retention, Intelligent-Tiering, compression |
| **Data transfer** | Cross-AZ chatty services, NAT for AWS service traffic, egress to internet/other clouds | VPC endpoints, AZ-aware routing (topology-aware routing in K8s), CDN for egress, compress, keep data and compute in the same region |
| **NAT gateway** | Image pulls, S3/ECR/logs through NAT | Gateway/interface endpoints, pull-through caches (topic 08) |
| **Observability** | Debug logs in prod, high-cardinality metrics, 100% traces, long hot retention | Sampling, drop/aggregate in pipeline, tiered retention, cardinality limits (topic 27) |
| **Load balancers** | One LB per service | Shared ingress/Gateway with host/path routing |
| **Idle/forgotten** | Old test environments, unused EIPs, orphaned resources | Tag policies, TTL tags on sandboxes, automated cleanup, Config rules |

### Pricing model strategy

```
          Usage over a month
   ▲
   │        peaks ─ Spot (fault-tolerant) / on-demand
   │   ┌──┐   ┌──┐
   │ ──┘  └───┘  └──── flexible part ─ on-demand (+ Spot)
   │ ██████████████████ steady baseline ─ Savings Plans / reservations (commit to ~70–80% of the minimum)
   └─────────────────────────────────▶ time
```
Commit to the **predictable floor**, not the peak; review coverage and utilization monthly.

### Data transfer cheat sheet (AWS, illustrative, check current pricing)

| Path | Typical charge |
|---|---|
| Internet → AWS (ingress) | Free |
| Within one AZ (private IP) | Free |
| **Cross-AZ** in a region | ~$0.01/GB **each direction** |
| **Cross-region** | ~$0.02/GB (varies by region pair) |
| **Internet egress** | ~$0.05–0.09/GB depending on volume tier (first 100 GB/month free across services) |
| **NAT gateway processing** | ~$0.045/GB + hourly |
| CloudFront to internet | Usually cheaper than EC2 egress at volume; origin → CloudFront free |

Example: 3 microservices exchanging 50 MB/s across AZs → 50 MB/s × 2.6 M s/month ≈ 130 TB/month × $0.02 (both directions) ≈ **$2,600/month** just for cross-AZ chatter. Topology-aware routing (prefer same-AZ endpoints) can cut much of it.

### Unit economics

```
Monthly cost of the checkout platform:   $42,000
Orders per month:                        6,000,000
Cost per 1,000 orders:                   $7.00
Track this monthly; alert if it rises > 15% without explanation.
CI example: $18,000/month ÷ 90,000 builds = $0.20 per build; ÷ 500 engineers = $36 per engineer
```
Unit costs make trade-offs concrete: "Moving logs to tiered storage saves $0.80 per 1,000 orders."

### Allocation: who spends what

- **Tagging policy:** `team`, `service`, `env`, `cost-center` required on all resources (enforced by IaC modules, tag policies, SCPs, admission policies for K8s labels).
- **Account per team/env** gives clean allocation automatically (topic 36).
- **Kubernetes:** shared clusters need allocation by namespace/labels: **OpenCost/Kubecost** split node costs by requested resources; shared costs (system pods, idle capacity) distributed proportionally.
- **Showback** monthly reports per team; **budgets and anomaly detection** alerts (AWS Budgets, Cost Anomaly Detection).

### Cost vs reliability vs speed

| Choice | Cost impact | Reliability impact |
|---|---|---|
| Multi-AZ (vs single AZ) | +10–50% (redundant capacity, cross-AZ traffic) | Big gain: survives AZ loss → **almost always worth it** for prod |
| Multi-region active-active | ~+100% | Survives region loss, latency gains → only for tier-0 / global needs |
| N+1 headroom | +30–50% capacity | Survives failures/spikes |
| Spot for stateless | −60–90% on that compute | Needs graceful interruption handling, diversification |
| Shorter log retention | Large savings | Less history for investigations/compliance → tier storage instead of deleting |
| Managed services | Higher per-unit price | Lower ops burden, often more reliable → usually cheaper in total (people cost) |

Senior framing: "**What's the cheapest design that meets the SLOs and RTO/RPO?**" and "**What would we gain for each extra $?**"

### FinOps practice

1. **Inform:** visibility, allocation, unit costs, forecasts.
2. **Optimize:** rightsizing, commitments, Spot, architecture changes, cleanup.
3. **Operate:** budgets, policies, cost reviews in design reviews and sprint rituals, accountability per team.

Cost appears in **design docs** (estimated monthly cost) and **PRs** (Infracost comments on Terraform plans, topic 36).

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Commitments | Savings Plans (flexible across instance families/regions for Compute SP) | Reserved Instances (specific, sometimes deeper discounts) |
| Managed vs self-hosted | Managed: higher unit price, low ops | Self-hosted: cheaper at scale, people cost and risk |
| Central vs team ownership of cost | Central FinOps team optimizes | Teams own their spend with central tooling ✅ (scales better) |
| Spot use | Aggressive (cheap, more interruptions) | Conservative (on-demand for critical) |

---

## Failure modes

| Failure | Symptom | Prevent |
|---|---|---|
| **Runaway autoscaling / bug** | Bill spike overnight | `maxReplicas`, budgets, anomaly alerts |
| **Forgotten resources** | Slowly rising baseline | Tag enforcement, TTL on sandboxes, cleanup automation |
| **Cost optimization breaks reliability** | Spot interruption kills single-replica service; rightsized DB saturates at peak | Optimize with SLOs in view; load test after rightsizing |
| **Over-commitment** | Paying for unused reservations after migration | Commit to floor, review monthly, shorter terms when uncertain |
| **Unallocated costs** | Nobody owns 40% of the bill | Required tags, account structure |
| **Log/metrics explosion** | Observability bill > app bill | Ingest quotas, sampling, cardinality limits |
| **Egress surprise** | Big data-transfer charges after an architecture change | Estimate transfer in design reviews; CDN; same-region placement |

**3 AM relevance:** cost anomalies are incidents too: a crypto-mining attacker on compromised credentials (topic 40) or a scaling loop can burn thousands per hour. Anomaly alerts should page for large, sudden spikes.

---

## Where you'll use it

- **Every case study's cost section**: name the top 2–3 cost drivers and levers.
- **CI/CD:** cost per build, Spot runners, caching (topic 47, [GHA 41](../../GitHub-Actions-Basics-To-Production/notes/41-billing-limits-and-cost.md)).
- **Logging platform** (topic 49): retention tiers are the cost design.
- **Kubernetes:** requests vs usage, Karpenter consolidation, Spot node pools (topic 38).
- **Terraform:** Infracost in PRs; tags enforced in modules.

---

## Design exercise

**Prompt:** A startup's AWS bill went from $40k to $95k/month in six months while customers grew 30%. Breakdown: EKS nodes $38k, RDS $14k, NAT gateway $11k, CloudWatch Logs $12k, data transfer $9k, S3 $4k, other $7k. Propose a plan.

<details>
<summary><b>Model answer</b></summary>

Unit cost check first: cost grew 2.4× vs customers 1.3× → cost per customer up ~80%. Leaks are likely.

1. **NAT $11k:** probably image pulls, S3, ECR and logs via NAT. Add S3 gateway endpoint (free), ECR/STS/Logs interface endpoints, ECR pull-through cache for public images. Expected −70–90% of NAT processing.
2. **CloudWatch Logs $12k:** find top log groups by ingestion; drop health-check/access-log noise, set DEBUG off in prod, set retention (14–30 days), ship bulk app logs to a cheaper pipeline (Loki/S3) and keep CloudWatch for AWS service logs. Expected −50–70%.
3. **EKS nodes $38k:** compare requests vs usage (often 2–3× over-requested); set requests from p95 usage, enable Karpenter consolidation, move stateless and CI workloads partly to Spot, adopt Graviton where images support ARM; schedule dev/staging down nights/weekends. Then buy a Compute Savings Plan for the new steady baseline. Expected −30–50%.
4. **Data transfer $9k:** identify cross-AZ chatter (VPC Flow Logs / Cost Explorer usage types); enable topology-aware routing for chatty services; check internet egress → CDN.
5. **RDS $14k:** rightsize using Performance Insights/CPU; check unused read replicas; reserved instances for the baseline; old manual snapshots cleanup.
6. **Process:** tagging enforced, showback per team, budgets + anomaly detection, Infracost in Terraform PRs, monthly cost review with unit cost per customer.

Expected outcome: ~$55–65k/month, with no SLO impact; verify each change with load tests/monitoring.
</details>

---

## Quick recap

- Cost is a requirement; state the main cost drivers of every design.
- Commit to the baseline, autoscale the variable part, use Spot for fault-tolerant work.
- Hidden costs: NAT processing, cross-AZ/region transfer, egress, logs/metrics, idle resources.
- Track unit cost (per request/customer/build) and allocate costs to owners via accounts and tags.
- Optimize within SLOs: cheapest design that meets reliability goals, not cheapest possible.

### Interview questions

1. **Q: Name some hidden cloud costs you've seen.**
   A: NAT gateway processing for traffic that should use VPC endpoints, cross-AZ data transfer between chatty services, internet egress without a CDN, CloudWatch/SaaS log ingestion from debug logs, high-cardinality metrics, orphaned volumes and snapshots, idle non-prod environments, and over-requested Kubernetes resources blocking bin-packing.

2. **Q: How do you reduce Kubernetes cost without hurting reliability?**
   A: Right-size requests from real usage, enable node consolidation, use Spot for stateless replicated workloads with PDBs and diversified instance types, Graviton where possible, scale non-prod down off-hours, buy Savings Plans for the steady floor, and keep headroom for N+1 per AZ. Validate with load tests and SLO dashboards after each change.

3. **Q: How would you make teams care about cost?**
   A: Give them visibility and ownership: account or namespace allocation, showback dashboards with unit costs, budget alerts routed to the team, cost estimates in design docs and Terraform PRs, and include cost in review of designs. Make efficient choices the default in platform templates.
