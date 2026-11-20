# System Design for DevOps / SRE / Platform Engineers: From Zero to Senior

> ✅ **Status: complete.** All 62 topics (Parts 0–10) written and self-reviewed, November 2026. Numbers in examples (prices, latencies, request rates) are **illustrative round numbers** for learning to estimate. Always check current cloud pricing and measure your own systems.

One learning path, in reading order. Each topic uses only ideas explained before it. When a later idea has to appear early, it is explained right there in a line or two, and a pointer like "(topic 24)" just tells you where the full story is. You never need to jump ahead to follow along.

**Every concept topic has two layers:**

1. **⚡ Core: the 10-minute version.** The idea in plain words, a picture to remember it by, the one thing to know, something to try, and a self-check.
2. **The full reference**, always in the same shape: What it is · Why it exists · Analogy · How it works inside (with a diagram and the request flow step by step) · Options and trade-offs (comparison table) · Failure modes (what breaks, how you detect it, how you fix or prevent it, and "3 AM") · Where you'll use it · Design exercise (with a model answer) · Quick recap + senior interview questions.

**Every case study (Part 9) has the interview shape:** Requirements and clarifying questions → Estimation → High-level design → Deep dives → Scaling → Reliability and DR → Security → Observability → Cost → Trade-offs → Follow-up questions with answers.

**Vendor-neutral first.** Each idea is taught as a concept, then shown on **AWS**, with the **GCP / Azure** equivalent in one line.

**Related notes:** [Linux](../linux-mastery-rhel/INDEX.md) · [Docker](../Docker/INDEX.md) · [Kubernetes](../CKA-Certification-Course-2025/INDEX.md) · [Jenkins](../Jenkins/INDEX.md) · [GitHub Actions](../GitHub-Actions-Basics-To-Production/notes/INDEX.md) · [Terraform](../Terraform/INDEX.md) · [Ansible](../Ansible/INDEX.md). Topics link straight to the matching topic file wherever a design builds on them.

**Levels:** ★ foundation · ★★ intermediate · ★★★ advanced (senior).

## How to study with these notes

| Your goal | Read | Time |
|---|---|---|
| **First pass: get the whole picture** | Only the **⚡ Core** of topics 01–44, in order | ~10 hours |
| **Become senior-ready** | Second pass: full reference + every design exercise (solve before reading the model answer) + every case study | A few weeks |
| **Interview tomorrow** | [01](notes/01-how-to-approach-system-design.md), [03](notes/03-back-of-the-envelope-estimation.md), [23](notes/23-availability-and-the-nines.md), [28](notes/28-sli-slo-sla-error-budgets.md), [41](notes/41-backups-and-disaster-recovery.md), two case studies, then [60](notes/60-interview-question-bank.md) + [62](notes/62-cheat-sheet.md) | An evening |
| **Production is on fire right now** | [58](notes/58-troubleshooting-at-scale.md) → [30](notes/30-incident-management.md) → [24](notes/24-resilience-patterns.md) | Minutes |

**How to do a design exercise:** cover the model answer, spend 10–15 minutes on paper (draw boxes, write numbers, list trade-offs), then compare. The model answer is *a* good answer, not the only one. If yours differs, check that you can defend your trade-offs.

---

# Part A · Foundations

## Part 0 · How to think

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 01 | ★ | [How to approach a system design interview](notes/01-how-to-approach-system-design.md) | The 45-minute plan, clarifying questions, functional vs non-functional requirements, driving the conversation, the common mistakes |
| 02 | ★ | [The map: a request's journey through a production system](notes/02-request-journey-map.md) | Client → DNS → CDN → load balancer → app → cache → database → queue as one picture that every later topic zooms into |
| 03 | ★ | [Back-of-the-envelope estimation](notes/03-back-of-the-envelope-estimation.md) | Powers of 2, latency numbers, DAU → QPS, peak factor, storage and bandwidth math, worked examples |

## Part 1 · Networking and the edge

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 04 | ★ | [Networking essentials for design](notes/04-networking-essentials.md) | IP, TCP vs UDP, ports, connections, latency vs bandwidth vs throughput, RTT, why distance costs time |
| 05 | ★ | [TLS and HTTPS](notes/05-tls-and-https.md) | Handshake cost, certificates and chains, where to terminate TLS, mTLS, certificate-expiry outages |
| 06 | ★ | [HTTP and API styles](notes/06-http-and-api-styles.md) | HTTP/1.1 vs 2 vs 3, REST vs gRPC vs WebSockets, status codes, idempotent methods, pagination, versioning |
| 07 | ★★ | [DNS for system design](notes/07-dns-for-system-design.md) | Resolution chain, record types, TTL trade-offs, weighted/latency/geo/failover routing, DNS as a single point of failure |
| 08 | ★ | [Cloud network building blocks](notes/08-cloud-network-building-blocks.md) | Regions, AZs, VPC, subnets, route tables, NAT, peering/transit, private endpoints; AWS with GCP/Azure equivalents |
| 09 | ★★ | [Load balancing](notes/09-load-balancing.md) | L4 vs L7, algorithms, health checks, connection draining, sticky sessions, global vs regional, ALB/NLB |
| 10 | ★★ | [Reverse proxies, API gateways and service discovery](notes/10-proxies-gateways-service-discovery.md) | Nginx/Envoy roles, gateway responsibilities, client- vs server-side discovery, how Kubernetes Services fit, service mesh in one page |

## Part 2 · Scaling compute

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 11 | ★ | [Scalability basics](notes/11-scalability-basics.md) | Vertical vs horizontal, stateless services, where sessions go, finding the bottleneck, Amdahl's warning |
| 12 | ★★ | [Autoscaling](notes/12-autoscaling.md) | Reactive, scheduled and predictive scaling, picking metrics, cooldowns, the HPA → nodes → ASG chain, scaling lag math |
| 13 | ★★ | [Monolith vs microservices](notes/13-monolith-vs-microservices.md) | Service boundaries, the cost of distribution, the modular monolith, when (not) to split |

## Part 3 · Caching and data

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 14 | ★★ | [Caching fundamentals and patterns](notes/14-caching.md) | Cache-aside, read/write-through, write-back, eviction, TTL, invalidation, stampede, hot keys, Redis vs Memcached |
| 15 | ★★ | [CDN and edge](notes/15-cdn-and-edge.md) | Edge caching, cache keys, origin shield, purging, static vs dynamic, edge compute, absorbing DDoS |
| 16 | ★★ | [Database fundamentals](notes/16-database-fundamentals.md) | SQL vs NoSQL families, ACID, indexes, query cost, connection pooling, how to choose |
| 17 | ★★★ | [Replication and failover](notes/17-replication-and-failover.md) | Leader-follower, multi-leader, leaderless, sync vs async, replication lag, failover, split brain |
| 18 | ★★★ | [Partitioning and sharding](notes/18-partitioning-and-sharding.md) | Range vs hash, consistent hashing, hot partitions, resharding, cross-shard queries |
| 19 | ★★★ | [Consistency, CAP and PACELC](notes/19-consistency-cap-pacelc.md) | Consistency models, quorums (R+W>N), what CAP really says, real database choices |
| 20 | ★★ | [Storage: block, file, object](notes/20-storage-block-file-object.md) | EBS/EFS/S3 and equivalents, durability vs availability, storage tiers, lifecycle rules |
| 21 | ★★ | [Queues, pub/sub and event streaming](notes/21-queues-pubsub-streaming.md) | Queue vs topic vs log, SQS vs SNS vs Kafka, ordering, consumer groups, DLQ, lag |
| 22 | ★★★ | [Async patterns that don't lose data](notes/22-async-patterns.md) | At-least-once + idempotency, the outbox pattern, sagas, why exactly-once is hard |

## Part 4 · Reliability foundations

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 23 | ★★ | [Availability and the nines](notes/23-availability-and-the-nines.md) | Nines → downtime math, serial vs parallel availability, redundancy, failure domains, finding SPOFs |
| 24 | ★★★ | [Resilience patterns](notes/24-resilience-patterns.md) | Timeouts, retries with backoff + jitter, circuit breakers, bulkheads, load shedding, graceful degradation, retry storms |
| 25 | ★★ | [Rate limiting and backpressure](notes/25-rate-limiting-and-backpressure.md) | Token bucket, leaky bucket, fixed/sliding windows, distributed limits, where to enforce, 429 vs 503 |
| 26 | ★★★ | [Distributed systems realities](notes/26-distributed-systems-realities.md) | Clocks, leader election, consensus (Raft/etcd), quorum loss, fencing, the fallacies of distributed computing |

---

# Part B · DevOps / SRE design

## Part 5 · Observability and SRE

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 27 | ★★ | [Observability: metrics, logs, traces](notes/27-observability.md) | Golden signals, RED/USE, percentiles, cardinality, sampling, OpenTelemetry, the cost of telemetry |
| 28 | ★★★ | [SLI, SLO, SLA and error budgets](notes/28-sli-slo-sla-error-budgets.md) | Choosing SLIs, setting SLOs, SLAs, error-budget policy, burn-rate math |
| 29 | ★★ | [Alerting and on-call design](notes/29-alerting-and-on-call.md) | Symptom vs cause alerts, multi-window burn-rate alerts, alert fatigue, runbooks, rotations |
| 30 | ★★ | [Incident management and postmortems](notes/30-incident-management.md) | Severities, roles, communication, mitigate before root-cause, blameless postmortems, action items that stick |
| 31 | ★★★ | [Capacity planning and performance testing](notes/31-capacity-planning-and-performance-testing.md) | Headroom, forecasting, load/stress/soak tests, Little's Law, why latency explodes near 100% utilization |

## Part 6 · Delivery

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 32 | ★★★ | [CI/CD platform design](notes/32-cicd-platform-design.md) | Pipeline architecture, runner fleets, caching, artifact flow, multi-tenancy, supply-chain security |
| 33 | ★★ | [Releases, promotion and GitOps](notes/33-releases-promotion-gitops.md) | Immutable artifacts, versioning, environment promotion, push vs pull (GitOps), rollback |
| 34 | ★★★ | [Deployment strategies](notes/34-deployment-strategies.md) | Recreate, rolling, blue/green, canary, feature flags, automated analysis and rollback, blast radius |
| 35 | ★★★ | [Database changes without downtime](notes/35-zero-downtime-database-changes.md) | Expand/contract, backward-compatible migrations, online schema change, backfills |

## Part 7 · Infrastructure and platform

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 36 | ★★★ | [IaC at scale](notes/36-iac-at-scale.md) | State layout and blast radius, module strategy, multi-account landing zones, policy as code, drift, Terraform + Ansible handoff |
| 37 | ★★★ | [Configuration and secrets at scale](notes/37-config-and-secrets-at-scale.md) | Config vs secrets, Vault/Secrets Manager, dynamic secrets, rotation, delivery into Kubernetes |
| 38 | ★★★ | [Kubernetes platform design](notes/38-kubernetes-platform-design.md) | One cluster vs many, multi-tenancy, node pools, ingress/gateway, autoscaling layers, upgrade strategy |
| 39 | ★★★ | [Platform engineering and internal developer platforms](notes/39-platform-engineering.md) | Golden paths, self-service, guardrails, measuring a platform (DORA metrics) |

## Part 8 · Security, DR and cost

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 40 | ★★★ | [Security architecture](notes/40-security-architecture.md) | Zero trust, IAM least privilege, segmentation, encryption, WAF/DDoS, supply chain, audit |
| 41 | ★★★ | [Backups and disaster recovery](notes/41-backups-and-disaster-recovery.md) | RPO/RTO, backup types, backup-restore → pilot light → warm standby → active-active, testing restores |
| 42 | ★★★ | [Multi-region architecture](notes/42-multi-region-architecture.md) | Active-passive vs active-active, cross-region data, traffic steering, failover/failback, cells |
| 43 | ★★ | [Cost-aware design and FinOps](notes/43-cost-and-finops.md) | Cost drivers (compute, storage, egress), pricing models, unit economics, showback, cost vs reliability |
| 44 | ★★ | [Chaos engineering and game days](notes/44-chaos-engineering-and-game-days.md) | Steady-state hypotheses, fault injection, blast-radius control, DR drills |

---

# Part C · Case studies and interview readiness

## Part 9 · Full interview-style designs

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 45 | ★★ | [Warm-up: a URL shortener](notes/45-case-url-shortener.md) | The whole framework on a small classic: estimation, key generation, cache, database, scaling |
| 46 | ★★★ | [A highly available web app on cloud](notes/46-case-ha-web-app.md) | Multi-AZ three-tier, autoscaling, managed DB, CDN, the 3 AM walkthrough |
| 47 | ★★★ | [A CI/CD platform for 500 engineers](notes/47-case-cicd-platform.md) | Runner fleet, queueing, caching, security, cost per build |
| 48 | ★★★ | [Zero-downtime deployments](notes/48-case-zero-downtime-deployments.md) | Canary + automated rollback + DB migrations + feature flags, end to end |
| 49 | ★★★ | [A central logging platform](notes/49-case-central-logging.md) | Agents → buffer → index → tiers, volume math, retention cost, multi-tenancy |
| 50 | ★★★ | [A metrics monitoring and alerting platform](notes/50-case-metrics-monitoring.md) | Prometheus at scale, remote write, long-term storage, alert routing, HA |
| 51 | ★★★ | [Multi-region Kubernetes](notes/51-case-multi-region-kubernetes.md) | Cluster per region, GitOps fleets, global traffic, stateful workloads, failover |
| 52 | ★★★ | [DR for a critical database](notes/52-case-database-dr.md) | Meeting a given RPO/RTO, replicas, PITR, cross-region copies, runbooks, drills |
| 53 | ★★★ | [Self-service infrastructure for 50 teams](notes/53-case-self-service-infrastructure.md) | Terraform modules, pipeline per state, policy, approvals, drift, guardrails |
| 54 | ★★★ | [A company-wide secrets management service](notes/54-case-secrets-management.md) | HA Vault, auth methods, dynamic credentials, unseal and DR, "Vault is down" |
| 55 | ★★★ | [A distributed rate limiter / API gateway](notes/55-case-rate-limiter-gateway.md) | Global vs local limits, Redis design, consistency trade-offs, fail open vs closed |
| 56 | ★★★ | [A distributed job scheduler](notes/56-case-job-scheduler.md) | Cron at scale, leader election, at-least-once runs, retries, observability |
| 57 | ★★★ | [A feature flag and config delivery service](notes/57-case-feature-flags-config.md) | Low-latency reads, streaming updates, safe rollout of config itself, outage behaviour |

## Part 10 · Interview readiness

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 58 | ★★★ | [Troubleshooting at scale](notes/58-troubleshooting-at-scale.md) | "It's slow", "502s", "DB CPU 100%", "one AZ is down": a method plus worked scenarios |
| 59 | ★★★ | [Reviewing someone else's design](notes/59-reviewing-designs.md) | The senior reviewer's checklist: SPOFs, hidden coupling, missing limits, cost traps |
| 60 | ★★ | [Interview question bank](notes/60-interview-question-bank.md) | Short strong answers to the common questions, grouped by topic |
| 61 | ★★★ | [Senior scenario interviews](notes/61-senior-scenario-interviews.md) | Open-ended design and debugging rounds with model answers and follow-ups |
| 62 | ★ | [Cheat sheet](notes/62-cheat-sheet.md) | Numbers, formulas, patterns and trade-off tables on one page |
