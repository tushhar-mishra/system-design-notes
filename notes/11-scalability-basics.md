# 11 · Scalability Basics

> **Level:** ★ foundation
>
> **You will learn:** what **scalability** means (and how it differs from **performance**), **vertical vs horizontal** scaling, why **stateless** services are the key to horizontal scale, where **state** goes (sessions, files, caches), how to **find the bottleneck**, **Amdahl's law** and the **Universal Scalability Law** in plain words (why adding servers stops helping), and the classic scaling path from one server to many.
>
> **You need:** topics 02–03, 09.
>
> **Part 2 · Scaling compute** · Previous: [10 · Proxies and gateways](10-proxies-gateways-service-discovery.md) · Next: [12 · Autoscaling](12-autoscaling.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Scalability** = the system can handle more load by adding resources, without a redesign. Two ways:
- **Vertical (scale up):** a bigger machine. Simple, but has a ceiling and is a single point of failure.
- **Horizontal (scale out):** more machines behind a load balancer. Near-unlimited, survives failures, but needs **stateless** services.

```
Stateful app (session in memory):        Stateless app (session in Redis):
User ─▶ LB ─▶ app-1 (has your cart)      User ─▶ LB ─▶ any app ─▶ Redis (your cart)
             app-2 (doesn't!)            Add/remove app servers freely
```

**Picture it.** A pizza shop. Vertical = buy a bigger oven. Horizontal = open more identical ovens. More ovens only work if any cook can make any order, which means the order slips are on a shared board (shared state), not in one cook's head.

**The one thing to know.** **Move state out of the app servers** (into databases, caches, object storage). Then app servers become cattle: identical, replaceable, autoscalable. The **hard** scaling problems then move to the stateful layer (topics 16–19).

**Try it now:** list everything your current app keeps on local disk or in memory between requests. Each item blocks horizontal scaling.

**You've got it if you can:** compare vertical and horizontal, list where each kind of state should live, and explain why doubling servers rarely doubles throughput.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **Performance** = how fast one request is (latency) at a given load.
- **Scalability** = how the system behaves **as load grows**. A scalable system keeps latency acceptable as you add resources in proportion to load.
- **Elasticity** = scaling up *and down* automatically with demand (topic 12).
- **Stateless service** = keeps nothing between requests in its own memory/disk that a later request needs. Any instance can serve any request.
- **Stateful service** = holds data that matters: databases, caches, queues, or apps with in-memory sessions.
- **Bottleneck** = the component that limits total throughput (the narrowest part of the bottle).

**"Pets vs cattle":** pets are unique servers you nurse back to health (named, hand-configured); cattle are identical, numbered, and replaced when sick. Horizontal scaling needs cattle. This is exactly what Docker images ([Docker 01](../../Docker/notes/01-why-containers.md)) and immutable infrastructure give you.

---

## Why it exists

Load grows (more users, more data, a marketing event). Without a plan, the system slows, then fails. Scalability thinking tells you **what to add** and **what will break next**, and keeps cost roughly proportional to load.

---

## Analogy

Pizza shop (Core), extended:
- **Vertical:** a bigger oven costs much more per pizza at the top end, and if it breaks, no pizza at all.
- **Horizontal:** more ovens, but now you need a counter to split orders (load balancer), a shared order board (shared state store), and a single cash register can become the next bottleneck (the database).
- **Amdahl's law:** if dough must be made by one person who takes 10% of every order's time, no number of ovens makes you more than 10× faster.

---

## How it works inside

### The scaling path (each step fixes the previous bottleneck)

```
Step 1  [app + DB on one VM]                                  ← simplest
Step 2  [app VM] → [DB VM]                                    ← separate concerns, scale each
Step 3  [LB] → [app ×N, stateless] → [DB]                     ← horizontal app tier
Step 4  ... → [cache] → [DB primary + read replicas]          ← offload reads (topics 14, 17)
Step 5  ... + [CDN] + [object storage for files]              ← offload static + bytes (15, 20)
Step 6  ... + [queue → workers]                               ← offload slow work (21)
Step 7  ... + [sharded/distributed DB]                        ← scale writes (18)
Step 8  ... + [multiple regions]                              ← global scale/DR (42)
```

Don't jump to step 7 for a step-3 problem. Each step adds cost and complexity.

### Where state goes

| State | Bad place (blocks scaling) | Good place |
|---|---|---|
| User sessions | App memory / local disk | Signed cookie / JWT (small), or Redis/DB session store |
| Uploaded files | Local disk of the instance | Object storage (S3) |
| Cache | Per-instance memory only (fine for tiny, rarely changing data) | Shared cache (Redis) or per-instance + short TTL |
| Background jobs | In-process threads | Queue + workers (topic 21) |
| Scheduled tasks (cron) | Cron on "the" server → runs N times on N servers | Single scheduler / K8s CronJob / leader election (topic 56) |
| Config | Edited files on servers | Env vars / ConfigMaps / config service (topic 37) |
| Logs | Local files | stdout → log pipeline (topic 49) |

The **Twelve-Factor App** guidelines (a well-known checklist for cloud-native apps) say the same: processes are stateless, config in the environment, logs as event streams, disposable processes (fast start, graceful stop).

### Finding the bottleneck

1. **Measure, don't guess.** Use the **USE method** for each resource: **U**tilization, **S**aturation (queueing), **E**rrors (topic 27). Use [Linux 034 · top and load average](../../linux-mastery-rhel/notes/034-top-and-load-average.md) and [Linux 103 · performance](../../linux-mastery-rhel/notes/103-performance-and-observability.md) on hosts.
2. Typical bottlenecks in order of frequency: **database** (slow queries, locks, connections), **downstream dependencies**, **thread/connection pools** in the app, **CPU** in the app, **network/NAT limits**, **disk I/O**.
3. After fixing one, **the next one appears**. Scaling is iterative.

### Why N servers ≠ N× throughput

- **Amdahl's law:** the part of work that can't be parallelised limits speed-up. If 5% is serial (e.g. a global lock, one DB row everyone updates), max speed-up is 1 / 0.05 = **20×**, no matter how many servers.
- **Universal Scalability Law** (plain words): beyond contention (waiting for shared things), there's **coherency** cost: servers coordinating with each other (cache invalidation, distributed locks, gossip). That cost grows with the *square* of the number of nodes, so past some point **adding nodes makes throughput go down**.
- **Design takeaway:** avoid shared hot spots (single counters, global locks), avoid all-to-all coordination, partition work so nodes don't need to talk (topic 18).

### Vertical scaling still matters

- Databases are often scaled vertically first, because it's the simplest way to scale a stateful system. Cloud instances go up to hundreds of vCPUs and terabytes of RAM.
- Vertical scaling usually needs a **restart** (downtime or failover), has **price jumps**, and has a **ceiling**.

---

## Options and trade-offs

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| How | Bigger instance | More instances + LB |
| Limit | Largest machine available | Mostly design (shared state, coordination) |
| Failure | Still one machine (SPOF) | Survives instance/AZ failure |
| Downtime to scale | Often a restart/failover | None (add instances) |
| App changes needed | None | Statelessness, externalized state |
| Cost curve | Super-linear at the top end | Roughly linear |
| Best for | Databases (first), legacy apps, quick fixes | Stateless web/API tiers, workers |

| State strategy for sessions | Pros | Cons |
|---|---|---|
| Sticky sessions | No code change | Uneven load, sessions lost when instance dies |
| Signed cookie / JWT | No server storage, scales infinitely | Size limits, revocation is hard (need short expiry or deny-list) |
| Redis session store | Simple, revocable | Another dependency; Redis must be HA |
| DB session table | Durable | DB load for every request |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Hidden state on instances** | Users randomly logged out, uploads "disappear" when hitting another instance | Errors after scale events/deploys | Externalize state; test by killing instances |
| **Cron on every instance** | Job runs N times (duplicate emails/charges) | Duplicate side-effects | Single scheduler, leader election, idempotent jobs |
| **Scaling the wrong tier** | Add app servers → DB melts faster | DB CPU/connections rise with app count | Measure bottleneck first; cache/replicas/pooling |
| **Hot spot** (one row/key everyone updates) | Throughput flat despite more servers; lock waits | DB lock metrics, per-key stats | Shard counters, batch, redesign (topic 18) |
| **Vertical scaling limit reached** | Biggest instance saturated | CPU/IO at max on largest size | Read replicas, partitioning, offload |
| **Slow start of new instances** | Scale-out doesn't help in time | Time from launch to ready | Faster images, warm pools, pre-scale (topic 12) |

**3 AM:** "We doubled the app pods, latency got worse." Classic sign the bottleneck is downstream (database, a dependency) and more pods means more concurrent load on it (more connections, more lock contention). Scale back, look at DB metrics, add caching/pooling, and rate-limit at the edge if needed.

---

## Where you'll use it

- **Kubernetes Deployments** are horizontally scaled stateless apps by design ([K8s 25](../../CKA-Certification-Course-2025/notes/25-deployments.md)); StatefulSets are for the hard stateful part ([K8s 40](../../CKA-Certification-Course-2025/notes/40-statefulsets.md)).
- **Docker:** immutable images make instances identical ([Docker 45 · production-ready containers](../../Docker/notes/45-production-ready-containers.md)).
- **Ansible / Terraform:** building cattle (ASGs with launch templates, baked images) instead of hand-configured pets.
- **Jenkins:** controller is a stateful pet (JENKINS_HOME); agents are cattle ([Jenkins 50 · agents](../../Jenkins/notes/50-agents-explained.md)).

---

## Design exercise

**Prompt:** A legacy PHP app runs on one big VM: Apache + PHP with sessions in local files, user uploads in `/var/www/uploads`, a nightly cron that emails reports, and MySQL on the same VM. Traffic is growing 3× next quarter. Plan the move to horizontal scale.

<details>
<summary><b>Model answer</b></summary>

1. **Separate the database:** move MySQL to a managed Multi-AZ instance (RDS). Scale it vertically for now; add a read replica if reads dominate.
2. **Externalize sessions:** switch PHP's session handler to Redis (ElastiCache, Multi-AZ).
3. **Uploads to object storage:** write to S3; serve via CDN with signed URLs if private. Migrate existing files with a sync job.
4. **Cron:** move to a single scheduled job (K8s CronJob, or EventBridge Scheduler → a task), make it idempotent (a "sent" marker per report) so a retry doesn't double-send.
5. **App tier:** bake a container image (or AMI), run ≥ 3 instances across 3 AZs behind an ALB, health check `/healthz`. Logs to stdout → central logging.
6. **Then** autoscale on CPU/requests (topic 12) and load test to find the next bottleneck (likely the DB → cache hot queries).

Order matters: do the state moves first (each is low-risk on its own and testable with 2 instances), then scale out. Trade-off: more moving parts (Redis, S3, RDS) and higher baseline cost, in exchange for surviving instance/AZ failures and scaling with traffic.
</details>

---

## Quick recap

- Scalability = handling more load by adding resources without redesign; performance = speed at a given load.
- Vertical is simple but limited and a SPOF; horizontal needs stateless services.
- Externalize state: sessions → Redis/cookies, files → object storage, jobs → queues, cron → one scheduler.
- Find the real bottleneck with measurement; it's usually the database.
- Serial parts and coordination cap scaling (Amdahl, USL): avoid hot spots and all-to-all chatter.

### Interview questions

1. **Q: What makes a service horizontally scalable?**
   A: It's stateless: no request depends on data held in a particular instance's memory or disk. Sessions, files, jobs and schedules live in shared systems. Instances are identical (same image, config from environment), start fast and shut down gracefully, so a load balancer and autoscaler can add and remove them freely.

2. **Q: We added servers but throughput didn't improve. Why?**
   A: The bottleneck is elsewhere, usually the database or a dependency, or there's a serialized part like a global lock or hot row (Amdahl's law). More app servers can even make it worse by adding connections and contention. I'd measure saturation per tier, then fix that tier (caching, pooling, query fixes, partitioning).

3. **Q: When would you choose vertical scaling?**
   A: For stateful components where scaling out is hard, typically a relational database, as the first step because it needs no app changes. Also for quick relief during an incident. I'd plan for its limits: a restart/failover to resize, price jumps, a ceiling, and it's still one machine, so pair it with a standby.
