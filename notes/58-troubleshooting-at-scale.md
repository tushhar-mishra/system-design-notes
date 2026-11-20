# 58 · Troubleshooting at Scale

> **Level:** ★★★ advanced
>
> **You will learn:** a repeatable **method** for debugging production systems (scope → timeline → what changed → follow the request → USE/RED → hypothesis → mitigate → confirm), the **questions that narrow a problem fast**, and **worked scenarios** that interviewers love: "it's slow", "502s at the load balancer", "database CPU at 100%", "one AZ is degraded", "latency only for some users", "Kubernetes pods pending", "intermittent DNS failures", "memory leak after deploy", "queue backlog", "cost spike".
>
> **You need:** topics 01–44 (this topic ties them together). Tool-level method: [Linux 102](../../linux-mastery-rhel/notes/102-troubleshooting-method.md), [K8s 81](../../CKA-Certification-Course-2025/notes/81-troubleshooting-method.md), [Docker 50](../../Docker/notes/50-debugging-method.md).
>
> **Part 10 · Interview readiness** · Previous: [57 · Feature flags](57-case-feature-flags-config.md) · Next: [59 · Reviewing designs](59-reviewing-designs.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The method (say it out loud in interviews):**

```
1. IMPACT & SCOPE   Who is affected? All users or some (region, AZ, endpoint, tenant, app version)? Since when?
2. WHAT CHANGED     Deploys, config/flag changes, infra (Terraform/K8s), traffic, dependencies, certificates, time (cron, month-end)
3. MITIGATE EARLY   If a recent change correlates → roll back. If overload → shed/scale. Don't wait for root cause.
4. FOLLOW THE REQUEST  Edge → DNS → CDN → LB → service → dependencies → data. Where does it go wrong first?
5. RED / USE        Services: Rate, Errors, Duration. Resources: Utilization, Saturation, Errors.
6. HYPOTHESIZE & TEST  One hypothesis at a time; confirm with data (traces, logs, metrics), not intuition
7. CONFIRM & PREVENT   SLIs recovered? Postmortem: prevent, detect, mitigate
```

**Picture it.** A plumber with a leak. First stop the water (mitigate), then ask "what changed?" (new pipe?), then follow the pipe from the main valve to the drip, checking pressure at each joint (follow the request + measure), not randomly tearing up the floor.

**The one thing to know.** **Scope is the fastest narrowing tool.** "Only users on Android app v5.2 in one region" tells you more than any dashboard. Slice by every dimension you have: region, AZ, endpoint, status code, version, tenant, node.

**You've got it if you can:** apply the method to an unseen scenario and talk through it in under 5 minutes with concrete commands/metrics.

> 📖 **That's the core.** The rest is worked scenarios.

---

## What it is

A **troubleshooting method** is a fixed sequence of questions that stops you from guessing. Under pressure, people jump to their favourite theory ("it's the network!"). A method makes you check impact, change history and the request path systematically, which is faster on average and easy to explain in interviews.

## Why it exists

In large systems, symptoms are far from causes (a slow DB shows up as 502s at the edge). MTTR (topic 23) depends mostly on how fast you narrow the search. Interviewers ask troubleshooting scenarios to see method, breadth (networking → Kubernetes → databases) and calm prioritization (mitigate first).

## Analogy

The plumber (Core). Also a detective: establish the facts (when, who, where), find what changed (motive and opportunity), test hypotheses against evidence, and don't arrest the first suspect.

---

## How it works inside: worked scenarios

### Scenario 1: "The site is slow"

```
Scope:    p99 latency up from 300 ms to 2 s since 14:10, all endpoints, all regions? → check slices
Changed:  deploy at 14:05 of catalog-svc; traffic normal
Follow:   CDN fine (edge latency OK) → ALB TargetResponseTime high → services: trace shows catalog-svc → DB span 1.5 s
USE (DB): CPU 95%, top query: new query without index from the 14:05 deploy
Mitigate: roll back catalog-svc → latency recovers
Prevent:  query plan checks in CI for big tables, slow query alert, canary analysis on latency
```
Key tools: dashboards sliced by endpoint/region, traces (find the slow span), `pg_stat_statements` / Performance Insights.

### Scenario 2: "502s at the load balancer"

| Clue | Likely cause | Check |
|---|---|---|
| 502s spike during deploys only | Pods killed mid-request, no graceful shutdown | Deploy timeline, preStop, SIGTERM handling (topic 34) |
| Constant low rate of 502s | Keep-alive timeout mismatch (backend closes idle connections before LB) | Backend keep-alive vs LB idle timeout (topic 06) |
| 502s + pod restarts | OOMKilled / crashes | `kubectl get pods`, `describe` → `OOMKilled`, logs `--previous` |
| 502s from one AZ/node | Node networking problem | Per-target error metrics |
| 502 with large responses/headers | Proxy buffer/header size limits | Ingress controller logs |
`502` = bad response from upstream; `503` = no healthy upstream/overload; `504` = upstream too slow (topic 06).

### Scenario 3: "Database CPU at 100%"

```
1. Top queries by total time (pg_stat_statements / Performance Insights / slow log)
2. New query? (deploy) → roll back / add index CONCURRENTLY
3. Same queries, more calls? → traffic spike or cache failure (hit ratio dropped? Redis failover?) → restore cache, shed load
4. Plan regression (stats changed)? → ANALYZE, compare plans, pin/fix
5. Lock contention / long transaction / vacuum issues? → pg_locks, oldest xact age
6. Batch job / analytics on primary? → move to replica, schedule off-peak
Mitigate fast: kill runaway queries, rate-limit the caller, scale up (failover blip), route reads to replicas
```

### Scenario 4: "One AZ is degraded"

```
Signals: errors/latency only from targets in ap-south-1b ; AWS Health event
Mitigate: shift traffic away (ALB zonal shift / ARC zonal shift, remove 1b subnets from LB, cordon nodes in 1b),
          ensure other AZs have capacity (scale up; Karpenter launches in a/c)
Check:    DB writer in 1b? (failover), single-AZ components (NAT, Redis primary, stateful pods with EBS in 1b)
Learn:    Was N+1 per AZ real? Static stability? (topics 23, 44)
```

### Scenario 5: "Latency only for some users"

Slice: by region/ISP (network/CDN PoP issue), by tenant (hot shard, noisy neighbour), by app version (client bug), by endpoint (one dependency), by user ID hash (one cache shard, one DB shard). Tools: RUM, CDN logs per PoP, per-shard metrics, per-tenant dashboards.

### Scenario 6: "Pods stuck Pending"

```
kubectl describe pod → Events:
  "0/40 nodes available: insufficient cpu"           → autoscaler? Karpenter logs, quotas, instance capacity, NodePool limits
  "node(s) had untolerated taint"                    → tolerations/affinity mismatch
  "volume node affinity conflict"                    → EBS volume in another AZ (topic 20)
  "pod has unbound immediate PersistentVolumeClaims" → StorageClass/CSI driver issue
  "exceeded quota"                                   → ResourceQuota in namespace
Also: subnet IPs exhausted (CNI errors in node/aws-node logs), admission webhook errors
```
([K8s 83 · data-plane troubleshooting](../../CKA-Certification-Course-2025/notes/83-data-plane-troubleshooting.md).)

### Scenario 7: "Intermittent DNS failures inside the cluster"

Symptoms: random `no such host` / 5 s delays. Causes: CoreDNS overloaded or throttled (CPU limits), `ndots:5` amplification, conntrack races with UDP, VPC resolver per-ENI packet limit. Fix: NodeLocal DNSCache, scale CoreDNS, FQDNs with trailing dot / lower ndots, connection reuse (topic 07).

### Scenario 8: "Memory keeps growing after a deploy"

```
Confirm: memory per pod rising linearly since deploy; OOMKilled restarts periodic
Mitigate: roll back (or increase restarts temporarily is NOT a fix, just buys time)
Diagnose: heap profile/dump from a canary pod (continuous profiling helps), compare versions
Prevent: soak tests, memory growth alert, canary bake time (topic 34)
```

### Scenario 9: "Queue backlog growing"

```
Is it producers (spike) or consumers (slow/erroring)?
  Producer rate ↑ → scale consumers (KEDA), check partition limit (Kafka: consumers ≤ partitions)
  Consumer errors → DLQ, logs: poison message? downstream failing (rate limit)? → backoff, fix, redrive
  Consumer slow → per-message latency ↑ → dependency slow (DB) → fix that
Check age of oldest message vs SLO and retention (data loss risk if retention expires) (topic 21)
```

### Scenario 10: "Cloud bill jumped 40% overnight"

Cost Explorer by service/usage type/tags: NAT bytes (new service pulling through NAT), data transfer (cross-AZ chatter), compute (runaway autoscaling or crypto-mining from compromised credentials → security incident!), logs ingestion (debug logging left on), S3 requests (loop). Treat unexplained compute spikes as potential security incidents (topic 40).

### Scenario 11: "Everything is down, nothing was deployed"

Check the "invisible" changes: certificate expiry, DNS/registrar, cloud provider incident, expired credentials/tokens (CI tokens, service account keys), quota limits, disk full, a scheduled job (month-end batch), a third-party dependency, clock/leap events, feature flag/config change, or an automated system (autoscaler, GitOps self-heal) that changed something.

---

## Options and trade-offs: tools by layer

| Layer | Tools / signals |
|---|---|
| Edge / DNS | `dig +trace`, synthetic checks, CDN logs/metrics, `curl -w` timings |
| Network | `mtr`, `tcpdump`, VPC Flow Logs, `ss -s`, conntrack, NAT metrics |
| Load balancer | 4xx/5xx by source (ELB vs target), target health, response time, surge queue |
| Kubernetes | `kubectl get/describe/logs --previous/events/top`, kube-state-metrics, node conditions |
| Application | RED dashboards, traces, structured logs by `trace_id`, profiles |
| Data | Slow query logs, locks, replication lag, cache hit ratio, queue lag |
| Changes | Deploy history (Argo CD), Terraform apply logs, flag change audit, CloudTrail |

---

## Failure modes (of troubleshooting)

| Anti-pattern | Fix |
|---|---|
| Debugging for an hour before mitigating | Roll back/shed first if a change correlates |
| Changing several things at once | One change, observe, announce |
| Trusting averages | Percentiles and slices |
| Ignoring "what changed?" | Change log/annotations always checked first |
| Looking only inside the app | Follow the whole request path, including DNS, certs, edge |
| Tunnel vision on a favourite theory | Write hypotheses, test each with data |

---

## Where you'll use it

Every on-call shift, every senior interview's troubleshooting round, every postmortem timeline.

---

## Design exercise

**Prompt:** At 09:30 on Monday, the support team reports "checkout is failing for some customers". Error rate is 4%. Walk through your first 15 minutes.

<details>
<summary><b>Model answer</b></summary>

1. **Declare/assess:** 4% checkout failures = SEV2 (SEV1 if rising). Open incident channel, IC (topic 30).
2. **Scope:** slice errors by region, AZ, endpoint, payment method, app version, tenant. Suppose: only card payments via provider A, all regions, started 09:12.
3. **What changed:** no deploys since Friday; provider A status page shows nothing yet; our flag changes: none; certificates: check provider A's client cert/mTLS cert expiry (Monday morning + nothing deployed → suspicious) → our client certificate for provider A **expired at 09:10**.
4. **Mitigate:** route card payments to provider B via flag (if available); renew/upload certificate (emergency process); verify error rate drops.
5. **Confirm:** checkout success SLI back to normal; reconcile failed attempts (customers may retry; idempotency protects from double charges).
6. **Follow-up:** expiry monitoring for all client certs (not only server certs), automated rotation, inventory of credentials with expiry dates (topic 05, 37).
</details>

---

## Quick recap

- Method: impact & scope → what changed → mitigate early → follow the request → RED/USE → hypothesis → confirm & prevent.
- Slice by every dimension; scope narrows fastest.
- Know the classic causes: deploys, config/flags, certs, DNS, capacity, dependencies, disk, quotas, credentials.
- 502/503/504 tell you different stories; Pending pods' events tell you exactly why.
- Mitigate first, root-cause later; one change at a time.

### Interview questions

1. **Q: Users say the app is slow but your dashboards are green. What do you do?**
   A: Confirm with user-side data (RUM, synthetic checks from their regions), then check what dashboards don't cover: DNS, CDN/edge, TLS, client-side, a specific segment (region, ISP, tenant, app version) hidden in averages. Slice percentiles by those dimensions and trace a slow request end to end.

2. **Q: How do you approach an outage with no recent deploys?**
   A: Look at invisible changes: certificates and credentials expiring, DNS/registrar, quotas, disk full, scheduled jobs, traffic patterns, provider incidents, config/flag changes, autoscaler/GitOps actions, third-party dependencies. Then follow the request path to find the first failing hop.

3. **Q: Database CPU is pegged and the site is failing. Walk me through it.**
   A: Mitigate (shed/rate-limit traffic, kill runaway queries, roll back a recent deploy), then look at top queries by total time to find the culprit: a new or regressed query, more calls due to cache failure, locks, or a batch job. Fix (index concurrently, restore cache, move batch to replica), confirm SLIs, and add prevention (plan checks, slow query alerts, cache resilience).
