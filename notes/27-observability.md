# 27 · Observability: Metrics, Logs, Traces

> **Level:** ★★ intermediate
>
> **You will learn:** the difference between **monitoring** and **observability**, the three main signals (**metrics, logs, traces**) plus **profiles** and **events**, what each is good and bad at, **percentiles** (why averages lie), the **four golden signals**, **RED** and **USE** methods, **cardinality** (the cost bomb of metrics), **structured logging**, **trace context propagation**, **sampling**, **OpenTelemetry**, correlating signals (**exemplars, trace IDs in logs**), **black-box vs white-box** monitoring, and the **cost** of telemetry.
>
> **You need:** topics 02, 03, 13, 23–24. [K8s 79](../../CKA-Certification-Course-2025/notes/79-monitoring-and-logging-basics.md) and [K8s 80](../../CKA-Certification-Course-2025/notes/80-production-observability.md) cover the K8s stack hands-on.
>
> **Part 5 · Observability and SRE** · Previous: [26 · Distributed systems](26-distributed-systems-realities.md) · Next: [28 · SLI/SLO/SLA](28-sli-slo-sla-error-budgets.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Observability** = being able to answer *new* questions about your system from the outside ("why are checkouts from Android slow since 14:05?") without shipping new code. You get it from telemetry:

```
Metrics  = numbers over time (request rate, error %, p99 latency, CPU). Cheap, fast, great for alerts and trends.
Logs     = timestamped event records ("order 9001 failed: card declined"). Rich detail; expensive at volume.
Traces   = the path of ONE request across services, with timing per step (spans). Find WHERE time/errors are.
```

**The four golden signals** (for any user-facing service): **Latency, Traffic, Errors, Saturation**.

**Picture it.** A car. Metrics = the dashboard gauges (speed, fuel, temperature). Logs = the trip diary ("10:42 engine warning light"). Traces = a GPS replay of one specific journey showing where you got stuck.

**The one thing to know.** Use **percentiles** (p50, p95, p99), not averages, for latency. An average of 100 ms can hide 1% of users waiting 5 seconds. And watch **cardinality**: a metric label like `user_id` creates millions of time series and can bring down your monitoring system.

**Try it now (PromQL):**
```promql
histogram_quantile(0.99, sum by (le) (rate(http_server_request_duration_seconds_bucket[5m])))
```

**You've got it if you can:** pick the right signal for a question, explain RED vs USE, and explain cardinality.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **Monitoring** = watching known things you decided in advance (dashboards, alerts on known failure modes).
- **Observability** = a property of the system: rich enough, connected telemetry that you can investigate *unknown* problems.
- **Telemetry** = the data emitted (metrics, logs, traces, profiles, events).

| Signal | Shape | Good for | Weak at | Tools |
|---|---|---|---|---|
| **Metrics** | Time series: name + labels → numbers | Alerting, dashboards, trends, SLOs, capacity | Details of individual requests | Prometheus, Mimir/Thanos, CloudWatch, Datadog |
| **Logs** | Text/JSON events | Debugging details, audit, errors with context | Aggregation at scale is costly | Loki, Elasticsearch/OpenSearch, CloudWatch Logs, Splunk |
| **Traces** | Tree of spans per request | Latency breakdown, dependency maps, cross-service debugging | Volume (need sampling) | Jaeger, Tempo, X-Ray, Zipkin, vendor APM |
| **Profiles** | CPU/memory per function over time | "Which code is burning CPU?" | Narrow | Pyroscope, Parca, cloud profilers |
| **Events** | Discrete changes (deploys, config changes, scaling) | Correlating "what changed?" | — | Annotations, change logs |

**OpenTelemetry (OTel)** = the vendor-neutral standard (CNCF) for instrumenting apps and collecting metrics, logs and traces: SDKs per language + the **OTel Collector** (receive → process → export to any backend). Instrument once, change backends freely.

---

## Why it exists

You can't fix what you can't see. In microservices, a single user request touches many services (topic 13); without correlated telemetry, incidents become guesswork and MTTR explodes (topic 23). SLOs (topic 28) and alerts (topic 29) are built on metrics. Capacity planning (topic 31) needs historical metrics.

---

## Analogy

Car dashboard / trip diary / GPS replay (Core). Extra: a **doctor** uses vital signs (metrics) to notice something's wrong, patient history (logs) to understand context, and an MRI (trace/profile) to locate the exact problem.

---

## How it works inside

### Metrics

```
http_requests_total{service="checkout", method="POST", route="/orders", status="500"}  12,345
          name                          labels (dimensions)                            value
```

Types (Prometheus):
- **Counter:** only goes up (requests, errors). Use `rate()` to get per-second.
- **Gauge:** goes up and down (memory, queue depth, temperature).
- **Histogram:** counts observations in buckets (latency ≤ 50 ms, ≤ 100 ms, ...) → percentiles computed at query time, aggregatable across instances ✅.
- **Summary:** percentiles computed in the client; **can't be aggregated** across instances ⚠️.

**Pull vs push:** Prometheus **scrapes** (pulls) `/metrics` endpoints; others receive **pushes** (OTLP, StatsD, CloudWatch agent). Pull gives "is the target up?" for free; push suits short-lived jobs and crossing networks.

### Percentiles: why averages lie

```
100 requests: 98 take 50 ms, 2 take 5,000 ms
Average = (98×50 + 2×5000)/100 = 149 ms   ← looks fine
p50 = 50 ms, p99 = 5,000 ms               ← 1 in 100 users waits 5 s
```
A user loading a page that makes 50 backend calls hits the backend p99 on **~40%** of page loads (1 − 0.99^50 ≈ 0.39). Tail latency matters.

⚠️ You **can't average percentiles** across instances (average of p99s ≠ global p99). Use histograms and aggregate buckets.

### Cardinality

**Cardinality** = number of unique time series = product of distinct values of each label.

```
http_requests_total{route (50), method (5), status (10), pod (200)} = 500,000 series
add user_id (1,000,000 users) → 500 billion → monitoring dies
```
Rules: never put unbounded values (user ID, request ID, email, full URL with IDs, IP) in metric labels. Those belong in **logs** or **traces**. Each active series costs memory in Prometheus (topic 03 exercise) and money in SaaS (often billed per series).

### The methods

| Method | For | Signals |
|---|---|---|
| **Four golden signals** (Google SRE) | User-facing services | Latency, Traffic, Errors, Saturation |
| **RED** (Rate, Errors, Duration) | Request-driven services (APIs, microservices) | Requests/s, failed/s, latency distribution |
| **USE** (Utilization, Saturation, Errors) | Resources (CPU, disk, network, pools, queues) | % busy, queue/wait length, error counts |

Use RED for every service dashboard; USE for every resource (node CPU, DB connections, thread pools, Kafka partitions). **Saturation** is the early warning: queues and waits grow before errors appear.

### Logs

- **Structured logs** (JSON) with consistent fields: `timestamp, level, service, env, trace_id, span_id, request_id, user_id (if allowed), message, error`. Machines can filter and aggregate them.
- Log to **stdout/stderr** in containers; an agent (Fluent Bit, Vector, OTel Collector) ships them ([Docker 33 · logging](../../Docker/notes/33-logging.md)).
- **Levels:** ERROR (needs attention), WARN, INFO (business events), DEBUG (off in prod, or sampled).
- ⚠️ **Never log secrets or sensitive personal data** (passwords, tokens, card numbers). Scrub in the app and in the pipeline.
- Cost: logs are usually the most expensive signal. Drop noise (health check access logs), sample DEBUG, tier storage (topic 49).

### Traces

```
Trace 4bf92f3577b34da6 (POST /checkout, 870 ms)
├─ gateway                      0–870 ms
│  └─ checkout-svc              5–865 ms
│     ├─ cart-svc GET /cart      10–40 ms
│     ├─ pricing-svc            45–95 ms
│     ├─ payments-svc           100–850 ms   ← 750 ms here
│     │  └─ provider HTTPS       110–845 ms   ← external API slow
│     └─ db INSERT order         852–860 ms
```
- **Span** = one operation with start, duration, attributes, status. **Trace** = all spans of one request.
- **Context propagation:** each service passes the trace ID in a header (**W3C `traceparent`**) to the next. Auto-instrumentation in OTel SDKs handles HTTP/gRPC/DB clients. Queue messages must carry the context too.
- **Sampling:**
  - **Head sampling:** decide at the start (keep 1–10%). Cheap; may miss rare errors.
  - **Tail sampling:** decide after the trace completes (keep all errors and slow traces + a % of normal) in the OTel Collector. Better signal, more resources.

### Correlation: making signals work together

```
Alert (metric: p99 latency high)
  → dashboard panel with EXEMPLARS (sample trace IDs attached to histogram buckets)
  → click a slow trace → see payments-svc span slow
  → jump to logs filtered by trace_id → "provider timeout after 750 ms, retrying"
  → events overlay: "payments-svc v2.14 deployed 14:03"
```
This flow (metrics → traces → logs → change events) is what cuts MTTR. Design for it: same labels (`service`, `env`, `version`) across all signals, trace IDs in logs, deploy annotations.

### Black-box vs white-box

- **White-box:** telemetry from inside (app metrics, logs).
- **Black-box / synthetic:** probes from outside like a user (Prometheus blackbox exporter, CloudWatch Synthetics, uptime checks from several regions). Catches DNS, certificate, CDN, and "everything is down so no metrics are emitted" failures.
- **RUM** (Real User Monitoring): telemetry from real browsers/apps (page load, errors).

---

## Options and trade-offs

| Choice | Option A | Option B |
|---|---|---|
| Stack | Self-hosted OSS (Prometheus, Grafana, Loki, Tempo, Mimir): cheaper at scale, you run it | SaaS (Datadog, New Relic, Grafana Cloud, Honeycomb): fast to start, costly at scale, per-series/GB pricing |
| Cloud-native | CloudWatch / Cloud Monitoring / Azure Monitor: integrated, easy for managed services | Less flexible querying, cost at volume |
| Instrumentation | OpenTelemetry (portable) ✅ | Vendor agents (lock-in, sometimes richer auto-instrumentation) |
| Logs | Full-text index (Elasticsearch): powerful search, expensive | Label index only (Loki): cheap, grep-like queries |
| Trace sampling | Head (cheap) | Tail (catches errors/slow, needs collector capacity) |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Cardinality explosion** | Prometheus OOM, huge SaaS bill | Series count growth, top metrics by series | Label rules, relabel/drop in scrape config, limits per tenant |
| **Monitoring down during incident** | Blind exactly when needed | Meta-monitoring | Monitor the monitoring from outside (dead man's switch, topic 29); HA |
| **Averages on dashboards** | Tail problems invisible | Users complain while graphs look fine | Percentiles, histograms, SLO-based views |
| **No correlation IDs** | Can't follow a request | Slow investigations | `trace_id` in logs, propagate context through queues |
| **Logging secrets/PII** | Compliance breach | Scanners, audits | Scrubbing, field allow-lists, access control on logs |
| **Log volume explosion** (debug left on, error loop) | Pipeline lag, cost spike, dropped logs | Ingest rate per service | Per-service rate limits/quotas in pipeline, sampling |
| **Sampling drops the one trace you need** | Can't find the failing request | Missing traces for errors | Tail sampling keeps errors |
| **Clock skew in logs** | Events appear out of order across hosts | Odd timelines | NTP; trust trace parent/child order over timestamps |
| **Dashboards nobody understands** | Slow triage | — | Standard RED/USE layouts per service, generated from templates |

**3 AM:** "Alert: checkout error rate 8%." Open the service RED dashboard → errors started 14:03 → deploy annotation at 14:02 (payments v2.14) → exemplar trace → errors from the new code path → **roll back first** (topic 30), investigate after.

---

## Where you'll use it

- Every design's "Observability" section (case studies): which SLIs, which dashboards, which alerts.
- **K8s:** kube-prometheus-stack, kube-state-metrics, node-exporter, OTel Collector as DaemonSet/Deployment ([K8s 80](../../CKA-Certification-Course-2025/notes/80-production-observability.md)).
- **CI/CD observability:** pipeline duration, queue time, failure rate (DORA metrics, topic 39) ([Jenkins 69](../../Jenkins/notes/69-monitoring-performance-ha.md)).
- **Docker:** container metrics/logs ([Docker 47 · monitoring](../../Docker/notes/47-monitoring-containers.md)).
- **Linux:** USE method on hosts ([Linux 103](../../linux-mastery-rhel/notes/103-performance-and-observability.md)).
- Platform case studies: logging (49), metrics (50).

---

## Design exercise

**Prompt:** A company with 60 microservices on EKS has only node CPU graphs and `kubectl logs`. Incidents take hours. Design an observability baseline for the next quarter (budget-conscious).

<details>
<summary><b>Model answer</b></summary>

1. **Standards first:** every service emits RED metrics (OTel SDK or framework middleware) with labels `service, route (templated, e.g. /orders/{id}), method, status_class, env, version`; JSON logs to stdout with `trace_id`; W3C trace context propagation including through SQS/Kafka message attributes.
2. **Collection:** OTel Collector (DaemonSet for logs and node metrics, gateway Deployment for traces with tail sampling: keep 100% errors, 100% of traces > 1 s, 5% of the rest). Prometheus (or managed Prometheus) scraping service and kube-state-metrics.
3. **Backends (OSS, cost-aware):** Mimir/Thanos or managed Prometheus for metrics, Loki for logs (label index, cheap object storage), Tempo for traces (object storage), Grafana for dashboards with exemplars linking metrics → traces → logs.
4. **Dashboards:** generated per service from a template (RED + saturation of its pools + dependencies), plus USE dashboards for nodes, DBs, queues.
5. **Black-box:** synthetic checks for top 5 user journeys from 3 regions; certificate expiry probes.
6. **Change events:** CI/CD posts deploy annotations (service, version, who) to Grafana.
7. **Guardrails:** cardinality limits per service, log ingest quotas per namespace, retention: metrics 30 d raw (13 months downsampled), logs 14 d hot / 90 d archive, traces 7 d.
8. **Then** SLOs and burn-rate alerts (topics 28–29) on the top user journeys.
Success measure: MTTD and MTTR in incident reviews.
</details>

---

## Quick recap

- Metrics for alerting/trends, logs for detail, traces for "where", profiles for "which code"; connect them with shared labels, trace IDs and exemplars.
- Golden signals; RED for services, USE for resources; saturation is the early warning.
- Percentiles from histograms, never averages of percentiles.
- Cardinality is the main metrics cost/risk: no unbounded label values.
- OpenTelemetry for vendor-neutral instrumentation; tail sampling keeps errors; synthetic checks see what users see.

### Interview questions

1. **Q: What's the difference between monitoring and observability?**
   A: Monitoring checks known conditions with predefined dashboards and alerts. Observability is the ability to ask new questions of the system using rich, correlated telemetry (high-dimensional metrics, structured logs, traces) so you can debug failures you didn't predict. You need monitoring for alerting and observability for investigation.

2. **Q: Why is putting `user_id` in a Prometheus label a bad idea?**
   A: Each unique label combination is a separate time series stored in memory. A label with millions of values multiplies series count, causing memory blow-ups, slow queries and big bills. Per-user detail belongs in logs or trace attributes; metrics stay low-cardinality.

3. **Q: How would you find which service makes checkout slow in a 40-service system?**
   A: Start from the checkout latency SLI, jump via exemplars to slow traces, read the span breakdown to find the slow service or dependency, then check that service's RED dashboard and logs filtered by trace ID, and overlay recent deploy/config events. Without tracing, I'd compare per-service latency dashboards along the call path, which is much slower.
