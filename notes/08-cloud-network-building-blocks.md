# 08 · Cloud Network Building Blocks

> **Level:** ★ foundation
>
> **You will learn:** **regions** and **availability zones** (the failure domains you design around), the **VPC** and how to plan its **CIDR**, **public vs private subnets**, **route tables**, **internet gateway**, **NAT gateway** (and its cost and AZ traps), **security groups vs NACLs**, connecting VPCs (**peering, Transit Gateway, PrivateLink**), **VPC endpoints**, hybrid links (**VPN, Direct Connect**), and the standard **three-tier VPC layout**, with AWS names and GCP/Azure equivalents.
>
> **You need:** topics 04, 07. [Terraform 29 · AWS basics](../../Terraform/notes/29-aws-basics.md) and [Terraform 32 · AWS networking](../../Terraform/notes/32-aws-networking.md) build these hands-on.
>
> **Part 1 · Networking and the edge** · Previous: [07 · DNS](07-dns-for-system-design.md) · Next: [09 · Load balancing](09-load-balancing.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A cloud is split into **regions** (separate geographic areas, e.g. Mumbai) and each region into **availability zones** (AZs: separate data centres with their own power and network, a few km apart). You build a private network (**VPC**) in a region, split it into **subnets** per AZ, and spread your servers across AZs so one data-centre failure doesn't take you down.

```
Region ap-south-1
 └─ VPC 10.0.0.0/16
     ├─ AZ-a: public 10.0.0.0/24 (LB, NAT)  private-app 10.0.10.0/24  private-db 10.0.20.0/24
     ├─ AZ-b: public 10.0.1.0/24             private-app 10.0.11.0/24  private-db 10.0.21.0/24
     └─ AZ-c: public 10.0.2.0/24             private-app 10.0.12.0/24  private-db 10.0.22.0/24
Internet ⇄ Internet Gateway ⇄ public subnets;  private subnets → NAT gateway (per AZ) → internet (outbound only)
```

**Picture it.** A region is a city. AZs are separate buildings in different neighbourhoods with their own generators. The VPC is your private campus spread over those buildings. Public subnets are the reception areas; private subnets are the offices behind badge doors.

**The one thing to know.** **Multi-AZ is the default for production.** A single-AZ design has an obvious single point of failure. Multi-region is a much bigger, more expensive step (topic 42).

**Try it now:** `aws ec2 describe-availability-zones --region ap-south-1 --query "AvailabilityZones[].ZoneName"`

**You've got it if you can:** draw the three-tier multi-AZ VPC, explain why NAT goes one per AZ, and pick between peering, Transit Gateway and PrivateLink.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Concept | AWS | GCP | Azure |
|---|---|---|---|
| Geographic area | Region | Region | Region |
| Isolated data centre(s) in a region | Availability Zone (AZ) | Zone | Availability Zone |
| Private network | **VPC** (regional) | **VPC** (**global**, subnets regional) | **VNet** (regional) |
| IP range slice | Subnet (in **one AZ**) | Subnet (**regional**, spans zones) | Subnet (regional) |
| Instance-level firewall | Security Group (stateful) | VPC firewall rules (stateful, by tag/SA) | Network Security Group (NSG) |
| Subnet-level firewall | NACL (stateless) | (firewall rules / hierarchical policies) | NSG on subnet |
| Internet access | Internet Gateway + public IP | Default internet route + external IP | Default / public IP |
| Outbound-only internet | NAT Gateway | Cloud NAT | NAT Gateway |
| Connect VPCs | VPC Peering, Transit Gateway, Cloud WAN | VPC Peering, Network Connectivity Center, Shared VPC | VNet Peering, Virtual WAN |
| Private access to a service | PrivateLink / VPC endpoints | Private Service Connect / Private Google Access | Private Link / Service Endpoints |
| Site-to-site VPN | Site-to-Site VPN | Cloud VPN | VPN Gateway |
| Dedicated line | Direct Connect | Cloud Interconnect | ExpressRoute |

Key terms:
- **CIDR block** = the IP range of the VPC/subnet (`10.0.0.0/16` = 65,536 addresses). AWS reserves 5 addresses per subnet.
- **Route table** = rules "traffic to destination X goes to target Y" attached to subnets.
- **Public subnet** = its route table sends `0.0.0.0/0` (everything) to an **Internet Gateway**.
- **Private subnet** = no direct internet route; outbound via a **NAT gateway** (or not at all).
- **Stateful firewall** = if a request is allowed in, the reply is automatically allowed out. **Stateless** = you must allow both directions explicitly.

---

## Why it exists

- **Failure isolation:** AZs fail independently (power, cooling, network, fires, floods). Regions fail independently of each other (rare, but it happens and is the reason for DR).
- **Security:** private subnets keep databases and apps unreachable from the internet. Defence in depth with security groups.
- **Latency:** AZs in a region are ~1 ms apart (cheap to replicate synchronously); regions are 10s–100s of ms apart.
- **Compliance:** data residency (keep data in a country's region).

---

## Analogy

| Cloud | Campus |
|---|---|
| Region | City |
| AZ | A building in a separate neighbourhood with its own power |
| VPC | Your private campus (fence around it) |
| Public subnet | Reception area with a street door |
| Private subnet | Offices accessible only from inside |
| Internet Gateway | The street door |
| NAT gateway | A mail room: staff can send letters out and get replies, but strangers can't walk in |
| Security group | A guard at each office door with a guest list |
| NACL | A checkpoint on the corridor, checking every person in both directions |
| Peering | A private footbridge to one other campus |
| Transit Gateway | A central bus station connecting many campuses |
| PrivateLink | A private window into one specific shop in another campus, nothing else |

---

## How it works inside

### The standard three-tier, multi-AZ VPC

```
                         Internet
                            │
                     [Internet Gateway]
                            │
   ┌────────────────────────┼─────────────────────────┐   VPC 10.0.0.0/16
   │   AZ-a                 │  AZ-b                     │
   │ ┌───────────────┐   ┌──┴────────────┐              │
   │ │ public subnet │   │ public subnet │   ← ALB nodes, NAT GW per AZ
   │ │ ALB  NAT-a    │   │ ALB  NAT-b    │
   │ └──────┬────────┘   └──────┬────────┘
   │ ┌──────▼────────┐   ┌──────▼────────┐
   │ │ private-app   │   │ private-app   │   ← EKS nodes / EC2 app servers
   │ │ (route 0/0 →  │   │ (route 0/0 →  │      outbound via NAT in SAME AZ
   │ │  NAT-a)       │   │  NAT-b)       │
   │ └──────┬────────┘   └──────┬────────┘
   │ ┌──────▼────────┐   ┌──────▼────────┐
   │ │ private-db    │   │ private-db    │   ← RDS primary (a) / standby (b)
   │ │ (no internet) │   │ (no internet) │      no route to internet at all
   │ └───────────────┘   └───────────────┘
   │   + VPC endpoints (S3 gateway endpoint, ECR/STS/Secrets Manager interface endpoints)
   └──────────────────────────────────────────────────────┘
```

**Request flow (inbound):** user → DNS → ALB public IPs in public subnets → ALB forwards to app targets' **private** IPs → app connects to DB in private-db subnet. Security groups: `alb-sg` allows 443 from the internet; `app-sg` allows 8080 only **from `alb-sg`**; `db-sg` allows 5432 only **from `app-sg`**. Referencing SGs instead of IP ranges is the key pattern.

**Outbound flow:** app pod calls an external payment API → route table sends `0.0.0.0/0` to NAT-a → NAT translates to its Elastic IP → Internet Gateway → internet. Replies come back through NAT.

**AWS service flow:** app pulls images from ECR and writes to S3. Without endpoints, that goes through NAT (paid per GB). With an **S3 gateway endpoint** (free) and **ECR interface endpoints**, it stays private and cheaper.

### CIDR planning

- Pick a VPC range that **won't overlap** with other VPCs, on-prem, or partners you may connect to later. Overlap blocks peering/TGW routing forever (re-IPing a VPC is painful).
- Use an **IPAM** (IP Address Management) plan: e.g. `10.0.0.0/8` split per environment and region: prod-ap-south-1 = `10.10.0.0/16`, staging = `10.20.0.0/16`.
- **Kubernetes eats IPs:** with the AWS VPC CNI, every pod gets a VPC IP. 100 nodes × 30 pods = 3,000 IPs. Give app subnets `/19` or bigger (8,192 addresses), or use prefix delegation / secondary CIDRs (e.g. from `100.64.0.0/10`).

### Connecting networks

| Option | Topology | Transitive? | Good for |
|---|---|---|---|
| **VPC peering** | 1:1 link | No (A↔B, B↔C ≠ A↔C) | Few VPCs, simple |
| **Transit Gateway** | Hub-and-spoke router | Yes | Many VPCs + VPN/Direct Connect; central inspection |
| **PrivateLink** (interface endpoint) | Consumer reaches **one service** via an ENI in its own VPC | N/A | Exposing a service to other accounts/VPCs without network-level access; overlapping CIDRs OK |
| **Shared VPC / RAM subnet sharing** | One network team owns the VPC; app accounts deploy into shared subnets | — | Central network control in multi-account orgs |

### Hybrid connectivity

- **Site-to-Site VPN:** encrypted tunnels over the internet. Minutes to set up, ~1.25 Gbps per tunnel on AWS, variable latency.
- **Direct Connect / Interconnect / ExpressRoute:** a dedicated physical link. Consistent latency, 1–100 Gbps, weeks to provision. For resilience: two connections at two locations, plus VPN as backup.

---

## Options and trade-offs

### NAT design

| Option | Cost | Resilience | Notes |
|---|---|---|---|
| **One NAT gateway per AZ** | ~3× hourly charge | AZ failure only affects that AZ | ✅ Production default |
| Single NAT gateway | Cheapest | ⚠️ NAT's AZ fails → all private subnets lose outbound; plus cross-AZ data charges | Dev/test only |
| NAT instances (EC2) | Cheaper per GB at high volume | You run and patch it; failover is yours | Rarely worth it |
| No NAT + VPC endpoints only | Cheapest, most secure | No general internet access | Locked-down workloads that only need AWS APIs |

NAT gateway data processing (roughly $0.045/GB in many AWS regions, illustrative) is a famous surprise bill: a cluster pulling images from Docker Hub or shipping logs to a SaaS through NAT can cost thousands per month. **VPC endpoints and caching registries** fix it (topic 43).

### Security groups vs NACLs

| | Security group | NACL |
|---|---|---|
| Applies to | ENI / instance / pod (with SG-for-pods) | Whole subnet |
| Stateful | Yes | No (allow return ephemeral ports!) |
| Rules | Allow only | Allow and deny, ordered |
| Use | Main control, reference other SGs | Coarse guardrails (block a range), rarely changed |

### Single region multi-AZ vs multi-region

| | Multi-AZ | Multi-region |
|---|---|---|
| Survives | Data-centre failure | Region failure |
| Data replication | Synchronous possible (~1 ms) | Usually async (RPO > 0) |
| Cost | ~1.1–1.5× | ~2× + data transfer + engineering |
| Complexity | Low (managed services do it) | High (topic 42) |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **AZ outage** | 1/3 of capacity gone; single-AZ components dead | Per-AZ health metrics, AWS Health events | Multi-AZ everything; capacity to run on N−1 AZs |
| **Single NAT in failed AZ** | All private subnets lose outbound (image pulls, external APIs fail) | NAT metrics, pull errors | NAT per AZ, route each AZ to its own NAT |
| **Subnet IP exhaustion** | Pods stuck `ContainerCreating` (CNI can't get IPs), ASG can't launch | `AvailableIpAddressCount` metric | Bigger subnets, prefix delegation, secondary CIDR |
| **Overlapping CIDRs** | Can't peer/route to a new VPC/partner | Design review | IPAM plan from day one |
| **SG rule too open** (`0.0.0.0/0` on 22 or 5432) | Security incident | Config rules, Security Hub, IaC scanners (topic 40) | Least privilege; SG references; policy as code |
| **NACL blocks ephemeral return ports** | Connections hang one way | VPC Flow Logs show REJECT | Allow 1024–65535 return traffic, or keep NACLs default |
| **NAT port exhaustion** | Outbound timeouts at high connection rates | `ErrorPortAllocation` | Connection reuse, more NAT IPs, endpoints (topic 04) |
| **Huge NAT / cross-AZ bill** | Surprise cost | Cost Explorer by usage type | Endpoints, pull-through caches, AZ-aware routing |

**3 AM:** "In one AZ, pods can't pull images; others fine." Check that AZ's NAT gateway / route table and any AZ-scoped endpoint. If the NAT is down and you only have one, you're down everywhere. That's why it's per AZ.

---

## Where you'll use it

- Every cloud design in Part 9 starts with this VPC.
- **Terraform:** the `terraform-aws-modules/vpc` module or your own ([Terraform 46 · public modules](../../Terraform/notes/46-public-modules.md)).
- **EKS:** subnet sizing, SG-for-pods, endpoints for a private cluster ([Jenkins 61 · AWS and K8s primer](../../Jenkins/notes/61-aws-and-k8s-primer.md)).
- **CI runners:** self-hosted runners in private subnets reaching AWS via endpoints (topic 47).
- **Cost reviews:** NAT and cross-AZ transfer are top line items (topic 43).

---

## Design exercise

**Prompt:** Design the network for an EKS-based platform: 3 environments (dev, staging, prod), each in its own AWS account, region `ap-south-1`, up to 200 nodes × 30 pods in prod, connection to an on-prem data centre, and shared services (CI runners, monitoring) in a separate "shared" account.

<details>
<summary><b>Model answer</b></summary>

- **IPAM plan:** `10.0.0.0/8` reserved for cloud. prod `10.10.0.0/16`, staging `10.20.0.0/16`, dev `10.30.0.0/16`, shared `10.40.0.0/16`. On-prem uses `172.16.0.0/12`; no overlap.
- **Per VPC:** 3 AZs; public `/24` per AZ (ALBs, NATs); private-app `/19` per AZ (prod pods: 6,000+ IPs needed, 3 × 8,192 gives headroom; plus prefix delegation); private-data `/24` per AZ.
- **NAT gateway per AZ** in prod/staging; single NAT in dev (accepted risk, saves cost).
- **VPC endpoints:** S3 gateway (free), interface endpoints for ECR (api + dkr), STS, Secrets Manager, CloudWatch Logs → most traffic avoids NAT.
- **Transit Gateway** in the network/shared account, attached to all four VPCs and to on-prem via **Direct Connect** (with Site-to-Site VPN as backup). TGW route tables: dev cannot route to prod; all can reach shared; only prod and shared reach on-prem.
- **Private EKS API endpoint**, reached from the shared account's CI runners via TGW.
- **DNS:** Route 53 private hosted zones shared across VPCs; Resolver endpoints for on-prem ↔ cloud name resolution.
- All built with Terraform, one state per account/layer (topic 36).

Trade-offs: TGW costs per attachment-hour and per GB processed, but peering doesn't scale (no transitivity, n² links). Dev single NAT: cheaper, but an AZ issue breaks dev outbound; acceptable for dev.
</details>

---

## Quick recap

- Region = city, AZ = isolated data centre; design multi-AZ by default, multi-region only when required.
- VPC with public, private-app and private-data subnets in each AZ; SGs reference each other.
- NAT gateway per AZ; use VPC endpoints to avoid NAT cost and keep traffic private.
- Plan non-overlapping CIDRs early; Kubernetes needs lots of IPs.
- Connect: peering (few), Transit Gateway (many), PrivateLink (one service); VPN/Direct Connect for hybrid.

### Interview questions

1. **Q: Why should each AZ have its own NAT gateway?**
   A: A NAT gateway lives in one AZ. If all private subnets route through it and that AZ fails, every AZ loses outbound internet, turning an AZ failure into a full outage. Also, cross-AZ traffic to a single NAT costs extra. Per-AZ NAT keeps failures contained.

2. **Q: When would you use PrivateLink instead of VPC peering?**
   A: When a consumer should reach **one service**, not the whole network: e.g. a platform team exposing an internal API to 50 accounts, or a SaaS vendor. It works with overlapping CIDRs, is one-directional, and avoids opening network-level routes. Peering gives full network reachability between two VPCs.

3. **Q: A cost review shows a huge NAT gateway bill. What do you check?**
   A: Which traffic goes through NAT: image pulls (ECR without endpoints, Docker Hub), S3 traffic without a gateway endpoint, logs/metrics shipped to a SaaS, cross-AZ hops to a single NAT. Fix with S3 gateway endpoint, interface endpoints for ECR/STS/logs, a pull-through cache for public images, and per-AZ NAT routing. VPC Flow Logs show the top talkers.
