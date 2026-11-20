# 42 · Multi-Region Architecture

> **Level:** ★★★ advanced
>
> **You will learn:** **why** (and when not) to go multi-region, **active-passive vs active-active**, the **data problem** (cross-region replication, write conflicts, home regions, global databases), **traffic steering** (DNS latency/geo routing, anycast global load balancers), **stateless vs stateful** tiers across regions, **cell-based architecture** for blast-radius control, **regional independence** (no cross-region calls on the hot path, static stability), **failover and failback**, **data residency**, and the **cost** (often 2×+).
>
> **You need:** topics 07, 09, 17–19, 23, 26, 41.
>
> **Part 8 · Security, DR and cost** · Previous: [41 · Backups and DR](41-backups-and-disaster-recovery.md) · Next: [43 · Cost and FinOps](43-cost-and-finops.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Running in more than one region gives **resilience to region failures**, **lower latency** for global users, and **data residency** options. The compute part is easy (copy the stack). The **data** part is hard: physics makes cross-region synchronous writes slow (50–250 ms), so you must choose who writes where.

```
Active-passive:   Users ─▶ Region A (serves all) ══async replication══▶ Region B (standby)
Active-active:    Users near A ─▶ Region A ⇄ data ⇄ Region B ◀─ Users near B
                  (each user/tenant has a HOME region for writes, or a multi-writer database)
```

**Picture it.** A company with two offices. Active-passive: everyone works in office A; office B is kept ready with copies of all files. Active-active: both offices work; but if both edit the same contract at the same time, you need rules (each client is owned by one office) or a merge process.

**The one thing to know.** **Each region must work on its own.** No synchronous calls to another region on the user's request path, no shared single control dependency. Otherwise you've built a system that fails when **either** region fails, which is *less* available than one region.

**Try it now:** list every dependency of your app. Mark which ones are regional, which are global (IAM, Route 53, a SaaS, a central auth server, a central DB). The global ones decide whether you're really multi-region.

**You've got it if you can:** choose active-passive vs active-active with reasons, design data placement (home region), and explain cells.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Pattern | Description |
|---|---|
| **Active-passive (hot/warm/pilot-light standby)** | One region serves; others ready to take over (topic 41) |
| **Active-active (read local, write home)** | All regions serve reads; each piece of data has one home region for writes |
| **Active-active (multi-writer)** | Any region writes any data; conflicts resolved by the database (LWW, CRDTs) or consensus (Spanner-style) |
| **Geo-partitioned** | Users/data pinned to a region (residency); regions mostly independent |
| **Cell-based architecture** | The system is split into many identical, independent **cells** (each a full stack serving a subset of users), within or across regions |

---

## Why it exists

1. **Region failures** happen (rare but real: large cloud regional incidents occur every year or two in some region). For tier-0 systems the business may require surviving them with RTO of minutes.
2. **Latency:** a user in São Paulo calling Mumbai pays ~300 ms RTT per round trip.
3. **Data residency/sovereignty:** laws requiring data to stay in a country/region (EU, India for some sectors, etc.).
4. **Scale and blast radius:** spreading load and limiting the impact of bad deployments (deploy region by region).

**When not to:** most systems. Multi-AZ in one region + good DR (topic 41) meets 99.95%+ for a fraction of the cost and complexity. Multi-region active-active roughly doubles infrastructure and greatly increases engineering complexity.

---

## Analogy

Two offices (Core). **Cells** = a franchise of many identical small shops, each serving its own neighbourhood. If one shop has a kitchen fire, only its neighbourhood is affected, and a bad new recipe is tried in one shop first.

---

## How it works inside

### Traffic steering

```
User ─▶ DNS (latency/geo routing + health checks)  ─or─  Anycast global LB (Global Accelerator / GCP global LB / Front Door / Cloudflare)
          │                                                   │
          ▼                                                   ▼
   nearest healthy region's ALB                       nearest healthy region's endpoint (seconds failover)
```
- DNS: cheap, failover in minutes, clients may cache (topic 07).
- Anycast/global LB: static IPs, fast failover, per-connection steering, costs more.
- **Sticky to home region:** for write-home designs, route the user to their home region (by user/tenant ID in a cookie/token, or the edge looks it up), or let any region accept the request and forward writes to the home region (adds latency for remote users).

### The data problem

| Approach | How | RPO on region loss | Write latency | Complexity |
|---|---|---|---|---|
| **Single primary region + async replicas** (active-passive) | Writes in A; read replicas in B | Seconds (lag) | Local in A; remote users pay RTT | Low–medium |
| **Home region per user/tenant** | Data partitioned by owner; each region primary for its users; replicated elsewhere for DR | Seconds for that partition | Local for home users | Medium (routing, moving users) |
| **Multi-writer with conflict resolution** (DynamoDB global tables, Cassandra multi-DC, Cosmos DB multi-region writes) | Any region writes; async replication; LWW or merge | ~seconds; conflicts possible | Local | Medium–high (app must tolerate conflicts) |
| **Globally consistent DB** (Spanner, CockroachDB, YugabyteDB, Aurora DSQL) | Consensus across regions | 0 | Cross-region quorum latency on writes (tens–hundreds of ms) | Lower app complexity, higher cost/latency |
| **Read-only/derived data everywhere** | Catalog, config, static content replicated | — | — | Low |

Typical real design: **classify data**:
- Reference data (catalog, config): replicate everywhere, eventual.
- User-owned data (profile, cart, orders): **home region**, replicated for DR.
- Global uniqueness / money / inventory: one authority (a region or a consensus DB) with the latency accepted for those operations.

### Regional independence (avoid cross-region coupling)

```
❌ Region B's app calls Region A's auth service on every request
   → A fails → B fails too; B's availability = A × B (worse than one region)
✅ Each region has its own auth (validating tokens locally), cache, queues, workers, observability
✅ Control planes (deploy, config) push to regions; regions keep running with last-known config if the control plane is down (static stability)
✅ Async replication between regions; never synchronous cross-region calls on the request path
```

Global services you still depend on (IAM, DNS, the CDN, your IdP, GitHub) should be on the **control plane** (needed to change things), not the **data plane** (needed to serve requests), wherever possible.

### Cell-based architecture

```
            [thin routing layer: user/tenant → cell]
            /        |         |         \
        cell-1    cell-2    cell-3    cell-4      (each = full stack: LB, apps, DB, cache, queue)
        (AZ-spanning, in region A)  (region B) ...
Each cell serves a fixed subset of customers; cells don't talk to each other.
Deploy: cell-1 (canary) → bake → cells 2..N in waves.
A bug or poison request affects one cell's customers, not all.
```
- **Cell size** bounded (e.g. ≤ 5% of customers) so blast radius is small and cell capacity is tested.
- The **routing layer** must be extremely simple and highly available (it's the shared part).
- Cells help with **noisy neighbours** too (move a big tenant to its own cell) and make capacity growth = "add a cell".
- Used widely by large SaaS and cloud providers (AWS publicly describes cell-based architecture as a resilience pattern).

### Failover and failback (active-passive)

```
Detect region problem → decide (human + data) → fence writes in old primary region (if reachable)
→ promote DB in new region → scale compute → shift traffic → verify
Failback later: rebuild old region as replica of new primary → planned switch back (or stay)
```
Avoid automatic regional failover on a single signal: false failovers are expensive and risk split brain. Automate the **steps**, keep a human **decision** for most systems (tier-0 may automate fully with careful quorum-based detection).

### Data residency

- Pin data (and its backups, logs, and derived copies!) to allowed regions.
- Watch the leaks: logs/traces shipped to a central region, analytics pipelines, support tooling, CDN caches.
- Geo-partitioned designs: EU users → EU cell; India users → India region.

### Cost of multi-region (rough)

| Item | Effect |
|---|---|
| Duplicate compute | Active-active: each region sized to take the other's load on failure → **each** at ~2× normal share |
| Data replication | Cross-region transfer (AWS ~$0.02/GB between many regions, illustrative) + replica storage |
| Global LB/DNS | Additional services |
| Engineering | Testing, tooling, data design, on-call complexity: often the largest cost |

---

## Options and trade-offs

| | Active-passive | Active-active (home region) | Active-active (multi-writer) |
|---|---|---|---|
| RTO | Minutes–hours (promotion, scaling) | Minutes (shift traffic) | Seconds–minutes |
| RPO | Seconds | Seconds (for failed region's data) | Seconds, with conflict risk |
| Latency for global users | Remote users pay RTT | Local for home users | Local |
| Capacity cost | Standby (cheaper, but must scale fast) | Each region sized for failover load | Same |
| App changes | Few | Routing by home, data partitioning | Conflict handling everywhere |
| Confidence in failover | Lower (standby path less exercised) | Higher (both regions always live) | High |

Senior rule: start **multi-AZ single region**; add **DR (pilot light/warm standby)** for tier-0/1; go **active-active** only when RTO/latency requirements and budget justify it, and do it with home regions or a database designed for it.

---

## Failure modes

| Failure | Symptom | Prevent |
|---|---|---|
| **Hidden cross-region dependency** | "Multi-region" app fails when one region fails | Dependency mapping, regional independence reviews, game days |
| **Failover capacity missing** | Surviving region overloaded → both down | Size each region for failover load (or pre-scaled warm capacity + quotas) |
| **Replication lag at failover** | Lost recent writes | Monitor lag; RPO alerts; reconciliation process |
| **Split brain (both regions write same data)** | Divergent data | Fencing, single home per record, careful promotion |
| **Write conflicts (multi-writer)** | Lost updates with LWW | Avoid concurrent writes to same item across regions; CRDTs; home-region writes |
| **Config/deploy pushed to all regions at once** | Global outage | Region-by-region waves with bake time |
| **Control plane global dependency** (IAM/STS endpoint, SaaS IdP) | Can't scale/log in during a regional event | Regional endpoints, static stability, cached tokens |
| **Data residency leak** | Compliance breach | Data flow inventory, region-pinned logging |
| **Untested failover** | Fails when needed | Regular drills, or active-active so both are always tested |

**3 AM:** "ap-south-1 is degraded." Active-active: shift traffic weights to the other region, confirm capacity scales, watch the replication backlog. Active-passive: follow the DR runbook decision criteria (topic 41); don't fail over for a 5-minute blip, since failover itself is risky.

---

## Where you'll use it

- **Case study 51** (multi-region Kubernetes) and **52** (database DR).
- **Terraform:** multi-region stacks with provider aliases and per-region state; replicated KMS keys, ECR replication.
- **GitOps:** one Argo CD per region, waves of promotion (topic 33).
- **Global services:** DNS (topic 07), CDN (topic 15).

---

## Design exercise

**Prompt:** A B2B SaaS (project management) has customers in India, EU and US. EU customers require data residency in the EU. Enterprise customers ask for 99.99% and RTO 15 min for regional failures. Design the regional architecture.

<details>
<summary><b>Model answer</b></summary>

- **Geo-partitioned, cell-based:** regions `ap-south-1` (India), `eu-central-1` (EU), `us-east-1` (US). Each tenant has a **home region** chosen at signup (EU tenants must be EU). Inside each region, **cells** (full stacks, ≤ ~5% of tenants each).
- **Routing:** a global, very simple routing layer: tenant subdomain (`acme.app.example.com`) → DNS/edge lookup → home region/cell. Routing table replicated to all edges; cached heavily.
- **Data:** each cell has its own Aurora cluster (Multi-AZ). For **regional DR** (RTO 15 min): each cell replicates to a **paired region within the allowed geography**: EU cells → `eu-west-1` (stays in EU), India → `ap-south-2` (Hyderabad), US → `us-west-2`. Aurora Global Database (RPO ~seconds), warm standby compute (Karpenter can scale from ~30%; quotas pre-raised).
- **Independence:** each region has its own auth token validation, queues, search index, observability stack (logs stay in-region for EU). Global pieces only on the control plane (deploys, tenant routing table).
- **Deploys:** cell canary → cells in waves within a region → next region, with bake times.
- **Failover:** per cell or per region via automated runbook (promote Global DB secondary, scale, flip routing for affected tenants). Human decision with a 5-minute decision SLA; automated steps ~8 minutes → RTO 15 min.
- **Cost:** warm standby in paired regions (~1.4–1.6× of the single-region cost), justified for enterprise tier; smaller tenants could be in cells with pilot-light DR to save cost (tiered offering).
- **Trade-offs:** tenant moves between cells/regions need a migration tool; cross-tenant features (global search across tenants) are limited by design.
</details>

---

## Quick recap

- Multi-region for region-failure resilience, latency and residency; most systems only need multi-AZ + DR.
- Compute is easy; data is hard: single primary, home region per user/tenant, multi-writer with conflicts, or a global consensus DB.
- Regions must be independent on the request path; global things belong on the control plane; static stability.
- Cells limit blast radius and noisy neighbours; deploy in waves.
- Size each region for failover load, test failover, and watch residency leaks and cost (≈2×).

### Interview questions

1. **Q: Active-active across regions: what's the hardest part?**
   A: Data. Cross-region synchronous writes are slow, so you either assign each record a home region (route writes there), accept multi-writer conflicts with a resolution strategy, or use a globally consistent database and pay write latency. Plus keeping regions independent, sizing each to absorb the other's load, and testing failover.

2. **Q: How can a multi-region design end up less available than a single region?**
   A: If regions depend on each other synchronously (a central auth service, a primary DB in one region on every request), the system fails when any of them fails, so availability multiplies down. Each region must serve requests on its own, with only async replication between them.

3. **Q: What is a cell-based architecture and why use it?**
   A: Splitting the system into many independent, identical full stacks (cells), each serving a subset of customers, with a thin router in front. Failures, bad deploys and noisy tenants are contained to one cell, capacity grows by adding cells, and each cell's size stays within tested limits.
