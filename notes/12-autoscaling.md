# 12 · Autoscaling

> **Level:** ★★ intermediate
>
> **You will learn:** **reactive, scheduled and predictive** autoscaling, choosing the **right metric** (CPU, requests per target, queue depth, custom), **target tracking vs step scaling**, **cooldowns** and **flapping**, the **scaling lag** math (why autoscaling can't save you from a sudden spike), the **Kubernetes autoscaling chain** (HPA → Cluster Autoscaler/Karpenter → cloud capacity), **KEDA** for event-driven scaling, **warm pools**, **scale-to-zero**, and what breaks.
>
> **You need:** topics 03, 09, 11. [K8s 48 · HPA](../../CKA-Certification-Course-2025/notes/48-metrics-server-and-hpa.md) and [K8s 49 · VPA and node autoscaling](../../CKA-Certification-Course-2025/notes/49-vpa-inplace-resize-node-autoscaling.md) cover the K8s objects.
>
> **Part 2 · Scaling compute** · Previous: [11 · Scalability basics](11-scalability-basics.md) · Next: [13 · Monolith vs microservices](13-monolith-vs-microservices.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Autoscaling** adds capacity when load rises and removes it when load falls, automatically. You pay for what you need, and survive daily peaks without manual work.

```
metric (e.g. CPU 80%) ──▶ autoscaler compares to target (60%) ──▶ desired = current × 80/60
          ▲                                                              │
          └──────────── new instances start, take traffic ◀──────────────┘   (this takes minutes!)
```

**Three kinds:**
- **Reactive:** respond to metrics now (target tracking). Default.
- **Scheduled:** "scale to 40 at 08:45 on weekdays". For known patterns.
- **Predictive:** forecast from history and scale ahead. For regular daily curves.

**Picture it.** A restaurant calling in extra waiters. Reactive = call when the queue at the door is long (they arrive 30 minutes later). Scheduled = always have extra staff on Friday night. Predictive = the manager looks at last month's Fridays and books staff in advance.

**The one thing to know.** **Autoscaling is slow** (new VM: 2–5 min; new K8s node + pod: 1–4 min; new pod on an existing node: seconds). So keep **headroom** (target ~50–70% utilization, not 90%), and for sudden spikes, **pre-scale** or **shed load** (topic 24).

**Try it now:** `kubectl get hpa -A` and read the `TARGETS` column: current vs target metric.

**You've got it if you can:** pick a scaling metric for a web API and for a queue worker, compute how much headroom you need for a spike, and explain the HPA → node autoscaler chain.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Autoscaler | Scales what | Based on |
|---|---|---|
| **AWS Auto Scaling Group (ASG)** / GCP Managed Instance Group / Azure VM Scale Set | VMs | CloudWatch metrics, schedules, predictive |
| **K8s HPA** (Horizontal Pod Autoscaler) | Pod replicas | CPU/memory (metrics-server), custom/external metrics |
| **K8s VPA** (Vertical Pod Autoscaler) | Pod CPU/memory requests | Observed usage |
| **Cluster Autoscaler** | K8s nodes (via ASGs/node groups) | Pending pods that don't fit |
| **Karpenter** | K8s nodes (directly launches right-sized instances) | Pending pods; also consolidation (bin-packing) |
| **KEDA** (Kubernetes Event-Driven Autoscaling) | Pods (incl. to/from zero) | Queue length, Kafka lag, cron, Prometheus queries... |
| **Serverless** (Lambda, Cloud Run, Azure Functions/Container Apps) | Function/container instances | Requests/events, automatically |
| **Managed DB/storage autoscaling** | Read replicas, storage, capacity units | Utilization (e.g. Aurora replicas, DynamoDB on-demand) |

**Target tracking** = "keep metric X at value Y"; the autoscaler computes how many instances are needed. **Step scaling** = "if CPU > 80% add 2; if > 90% add 5". Target tracking is simpler and usually better.

**HPA formula:**
```
desiredReplicas = ceil( currentReplicas × currentMetricValue / targetMetricValue )
e.g. 10 pods at 90% CPU, target 60% → ceil(10 × 90/60) = 15 pods
```

---

## Why it exists

Traffic varies: daily cycles (often 3–10× between night and peak), weekly cycles, events. Fixed capacity for peak wastes money most of the day; fixed capacity for average fails at peak. Autoscaling also **replaces** failed instances (ASG health checks), which is part of availability.

---

## Analogy

Restaurant staffing (Core). Extras:
- **Cooldown** = after calling in waiters, wait to see the effect before calling more (or sending them home).
- **Flapping** = sending waiters home, then immediately calling them back, all evening.
- **Warm pool** = waiters already dressed and waiting in the staff room: they start in 1 minute instead of 30.

---

## How it works inside

### The Kubernetes chain (each layer must react in time)

```
Traffic ↑
  │
  ▼  ~15–60 s (metrics scrape + HPA sync every 15 s)
[HPA] raises replicas 10 → 15
  │
  ▼  seconds if room on nodes; else pods "Pending"
[Scheduler] places pods ─── no room? ──▶ [Cluster Autoscaler / Karpenter] sees Pending pods
                                             │  ~1–3 min: launch EC2, boot, join cluster,
                                             ▼  CNI ready, image pull
                                          new node Ready → pods scheduled
  │
  ▼  + app startup time (JVM warm-up, cache load): 10 s – 2 min
  ▼  + readiness probe passes → added to Service endpoints / LB targets
New capacity serving traffic: total ~30 s (room available) to ~3–5 min (new nodes)
```

**HPA needs resource requests** set on pods (CPU% is relative to the request) ([K8s 41](../../CKA-Certification-Course-2025/notes/41-requests-limits-qos-quotas.md)).

### Scaling lag math: how much headroom?

Say traffic can rise **from 1,000 to 1,600 req/s within 3 minutes** (60% in 3 min), and new capacity takes **4 minutes** to serve.

```
Each pod handles 100 req/s comfortably (measured).
At 1,000 req/s with target 60% utilization → pods = 1,000 / (100 × 0.6) ≈ 17 pods
Capacity of 17 pods at 100% = 1,700 req/s
The spike to 1,600 arrives before new pods → 17 pods must absorb it: 1,600/1,700 = 94% → OK but tight
With target 80%: pods = 13 → capacity 1,300 < 1,600 → overload during the lag ✗
```

**Rule:** target utilization ≈ 1 / (expected spike factor within the scaling lag). Expect +50% within the lag → target ~65%.

### Choosing the metric

| Workload | Good metric | Why |
|---|---|---|
| Stateless web/API | **Requests per target** (ALB `RequestCountPerTarget`) or CPU | Directly tied to load |
| CPU-bound service | CPU utilization | Simple, built in |
| Queue workers | **Queue depth per worker** or **age of oldest message** (KEDA, SQS metrics) | CPU may be low while backlog grows (I/O-bound workers) |
| Kafka consumers | **Consumer lag** | Max useful replicas = number of partitions! (topic 21) |
| Latency-sensitive | p95/p99 latency (careful) | Latency rises for many reasons (e.g. DB slow); scaling won't help them |
| Memory-bound | ⚠️ Memory rarely good for HPA | Many runtimes don't release memory → never scales down |

### Scale-down behaviour

- Scale **up fast, down slow**: HPA `behavior` defaults to a 5-minute stabilization window for scale-down.
- Scale-down must be **graceful**: connection draining, finish in-flight work, workers finish/requeue the current message (topic 09, 21).
- Node scale-down (Karpenter consolidation / CA) evicts pods: protect with **PodDisruptionBudgets** and don't run single replicas of important things.

### Scheduled and predictive

- **Scheduled:** `minReplicas`/`desiredCapacity` raised before known peaks (office hours, sales, TV ads, end-of-month batch). KEDA `cron` scaler or ASG scheduled actions.
- **Predictive:** AWS predictive scaling for ASGs uses up to 14 days of history to forecast. Works for regular cycles, not one-off events.

### Warm pools and fast starts

- **ASG warm pools:** pre-initialized stopped/hibernated instances → start in seconds.
- **Overprovisioning in K8s:** low-priority "placeholder" pods reserve capacity; real pods preempt them instantly, and the autoscaler adds nodes for the evicted placeholders ([K8s 47 · priority and preemption](../../CKA-Certification-Course-2025/notes/47-priority-preemption-eviction.md)).
- **Faster images:** smaller images, pre-pulled images, lazy loading (SOCI/stargz), faster app startup.

---

## Options and trade-offs

| Strategy | Pros | Cons | Use when |
|---|---|---|---|
| Reactive target tracking | Simple, adapts to anything | Lags behind sudden spikes | Default for everything |
| Step scaling | Fine control | More tuning, more ways to get it wrong | Special cases |
| Scheduled | Ready before known peaks | Wasted if the peak doesn't come; manual calendar | Office hours, known events |
| Predictive | Pre-scales regular curves | Needs history; misses anomalies | Strong daily/weekly patterns |
| Overprovisioning / warm pools | Near-instant capacity | Pays for idle capacity | Spiky, latency-critical services |
| Scale to zero (KEDA, serverless) | No cost when idle | Cold-start latency | Dev envs, batch, rare events |
| Fixed capacity | Predictable, no scaling bugs | Pays for peak always | Stateful systems, small steady load |

**Cluster Autoscaler vs Karpenter** (EKS): CA scales pre-defined node groups (simple, predictable instance types); Karpenter picks instance types per pending pods, consolidates underused nodes, and handles Spot well (more efficient, but more churn; needs PDBs and disruption budgets).

**Spot / preemptible instances** = spare capacity at large discounts (often 60–90%) that the cloud can take back with ~2 minutes notice (AWS). Great for stateless, fault-tolerant workers and CI runners, with diversification across instance types and AZs. Not for databases or single replicas (topic 43).

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Scaling too slow for spike** | Errors/latency during first minutes of a spike | Saturation before new capacity ready | Lower target, pre-scale, warm pools, load shedding |
| **No requests set** | HPA shows `<unknown>` and never scales | `kubectl describe hpa` | Set CPU requests; install metrics-server |
| **Max replicas too low** | Pinned at max while overloaded | HPA at `maxReplicas` alert | Raise max, check quotas |
| **Cloud capacity/quota limits** | Pods pending; ASG "insufficient capacity" | Pending pods, ASG activity errors | Diversify instance types/AZs, raise service quotas in advance |
| **Subnet IP exhaustion** | New nodes/pods fail | IP availability metric | Bigger subnets / prefix delegation (topic 08) |
| **Flapping** | Constant scale up/down, churn | Replica count graph sawtooth | Stabilization windows, cooldowns, wider target |
| **Scaling on the wrong metric** | Workers idle on CPU while queue grows | Queue age rising, replicas flat | Scale on queue depth/lag |
| **Scaling amplifies a downstream problem** | DB slow → latency up → HPA adds pods → more DB load | Pods ↑ while DB saturation ↑ | Don't scale on latency alone; protect downstream with pools/limits |
| **Scale-down kills work** | Lost jobs, 5xx | Errors during scale-in | Graceful termination, PDBs, drain, idempotent jobs |
| **Runaway scaling (bug/attack)** | Huge bill | Cost anomaly alerts | `maxReplicas`, budgets, rate limiting at edge |

**3 AM:** "HPA at max, pods Pending, latency high." Check: Karpenter/CA logs (can't launch? quota? capacity?), subnet IPs, Spot interruptions. Immediate: raise limits or add on-demand capacity; shed non-critical traffic. Afterwards: capacity planning for the peak (topic 31).

---

## Where you'll use it

- **K8s:** HPA for Deployments, KEDA for queue workers, Karpenter for nodes.
- **CI runners:** ephemeral runners scaled on queued jobs (Actions Runner Controller, [GHA 38](../../GitHub-Actions-Basics-To-Production/notes/38-self-hosted-runners-deep-dive.md); Jenkins Kubernetes agents, [Jenkins 53](../../Jenkins/notes/53-kubernetes-and-cloud-agents.md)).
- **Terraform:** `aws_autoscaling_group`, `aws_autoscaling_policy` (target tracking), scheduled actions.
- **Cost:** scale down at night and in non-prod (topic 43).

---

## Design exercise

**Prompt:** An image-processing service: users upload photos; a worker resizes each in ~2 s of CPU. Uploads average 20/s, but newsletters cause bursts of 300/s for 10 minutes. Users should see thumbnails within 2 minutes. Design the scaling.

<details>
<summary><b>Model answer</b></summary>

- **Decouple:** uploads go to S3; an event goes to a queue (SQS). Workers pull from the queue (topic 21). The queue absorbs the burst, so the requirement becomes "drain within 2 minutes", not "process instantly".
- **Math:** each worker (1 vCPU) does 0.5 images/s. Average 20/s → 40 workers. Burst 300/s → 600 workers to keep up in real time. But we only need **latency < 2 min**: in a 10-minute burst of 300/s = 180,000 images. To finish each within ~2 min of arrival, throughput must roughly match arrival rate (otherwise backlog grows), so ~600 workers during the burst, or accept longer delay.
- **Scaling:** **KEDA** on SQS with a target of ~(messages per worker) and also the **age of oldest message** (alert if > 90 s). Min 40 workers, max 700. Nodes from **Karpenter** with **Spot** instances (workers are idempotent and restartable; diversified instance types).
- **Scheduled pre-scale:** newsletters are sent by us, so the marketing tool triggers a pre-scale to 400 workers 5 minutes before sending (or a KEDA cron trigger).
- **Graceful:** visibility timeout > processing time; on SIGTERM (Spot interruption), the worker stops taking new messages; unfinished messages return to the queue. Processing is idempotent (same output key overwrite).
- **Cost:** Spot keeps burst cost low; scale back down after the queue drains (stabilization 2–3 min).
</details>

---

## Quick recap

- Reactive (default), scheduled (known peaks), predictive (regular cycles).
- Scale on a metric that tracks load: requests per target, CPU, queue depth/lag; not memory, careful with latency.
- Autoscaling lags by minutes: keep headroom (target ≈ 1 / spike factor), pre-scale, warm pools, shed load.
- K8s chain: HPA → pending pods → Karpenter/CA → new nodes → readiness. Each step can block.
- Scale down slowly and gracefully; cap max replicas and watch cost.

### Interview questions

1. **Q: Your service gets a 3× spike within 1 minute when a push notification goes out. Will autoscaling handle it?**
   A: Not on its own. New pods/nodes take one to several minutes. Options: pre-scale before sending (we control the trigger), keep more headroom, overprovision with placeholder pods or warm pools, stagger the notifications, and have load shedding/rate limiting so the spike degrades gracefully instead of cascading.

2. **Q: What metric would you autoscale Kafka consumers on?**
   A: Consumer lag (messages behind) per partition, via KEDA. Note the ceiling: in a consumer group, max useful consumers = number of partitions; beyond that, extra pods sit idle. So partition count must be planned for peak parallelism.

3. **Q: HPA scaled up, but latency didn't improve. Why?**
   A: Either pods are Pending (no node capacity, quota, IPs) or the bottleneck is downstream (database, external API, a lock), so more pods add more load to it. I'd check pending pods and node autoscaler events, then saturation of dependencies. Scaling on latency is risky for exactly this reason.
