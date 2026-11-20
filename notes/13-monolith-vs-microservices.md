# 13 · Monolith vs Microservices

> **Level:** ★★ intermediate
>
> **You will learn:** what **monoliths**, **modular monoliths** and **microservices** are, the **real costs** of distributing a system (network failures, data consistency, operations, observability), how to draw **service boundaries** (by business capability, by data ownership), **Conway's law**, the **distributed monolith** anti-pattern, **sync vs async** communication between services, the **strangler fig** migration, and what microservices mean for your **CI/CD, Kubernetes and on-call**.
>
> **You need:** topics 06, 10, 11. [K8s 05 · app architecture](../../CKA-Certification-Course-2025/notes/05-app-architecture-monolith-3tier-microservices.md) is the short intro.
>
> **Part 2 · Scaling compute** · Previous: [12 · Autoscaling](12-autoscaling.md) · Next: [14 · Caching](14-caching.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.**
- **Monolith:** one deployable app containing all features, usually one database.
- **Modular monolith:** still one deployable, but with strict internal modules and boundaries.
- **Microservices:** many small services, each owning one business capability **and its own data**, deployed independently, talking over the network.

```
Monolith                     Microservices
[ users | orders | payments ]   [users]──▶[users DB]
         │                      [orders]──▶[orders DB]   ← each owns its data
     [ one DB ]                 [payments]──▶[payments DB]
                                 (talk via APIs / events)
```

**Picture it.** A monolith is a big department store: one building, one management, easy to walk between departments. Microservices are a street of independent shops: each can open late, renovate or hire on its own, but now customers walk outside between shops (network), and shops must coordinate deliveries (consistency).

**The one thing to know.** Microservices solve an **organizational** scaling problem (many teams stepping on each other) at the price of a **technical** one (distributed systems). If you don't have the team-scaling problem, you mostly get the price.

**Try it now:** for an app you know, list which features change together and which teams own them. Those clusters hint at service boundaries.

**You've got it if you can:** argue both sides with trade-offs, spot a distributed monolith, and describe the strangler fig migration.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Style | Deploy unit | Data | Team fit |
|---|---|---|---|
| **Monolith** | One | One shared DB | 1–2 teams |
| **Modular monolith** | One | One DB, but each module owns its tables, no cross-module table access | Several teams, one codebase |
| **Microservices** | Many, independent | Database per service | Many autonomous teams |
| **SOA** (Service-Oriented Architecture, 2000s) | Several big services, often with an ESB (Enterprise Service Bus) | Often shared | Enterprise integration |
| **Serverless functions** | Per function | Varies | Event-driven pieces |

**Service boundary** = the line between what one service owns (code + data + API) and what it asks others for. Ideally drawn around a **business capability** (orders, payments, inventory), a concept from **DDD** (Domain-Driven Design) called a **bounded context**.

**Conway's law:** systems end up mirroring the communication structure of the organisations that build them. Teams build services; so design team boundaries and service boundaries together.

---

## Why it exists

Monolith pains at scale (many engineers):
- One deploy for everything → release trains, merge conflicts, one bad change blocks all.
- One tech stack, one scaling unit (can't scale only the image-resizer).
- Long builds and test suites.
- A bug in one feature (memory leak) can crash everything.

Microservices answer: independent deploys, independent scaling, fault isolation, team autonomy.

Their costs (often underestimated):
- **Every call is a network call**: latency, timeouts, retries, partial failures (topic 24).
- **No shared transactions**: consistency across services needs sagas/outbox (topic 22).
- **Operations multiply**: N pipelines, N dashboards, N on-call surfaces, versioning, contracts.
- **Debugging needs distributed tracing** (topic 27).
- **Platform needed**: K8s, mesh/gateway, CI/CD templates (topics 32, 38, 39).

---

## Analogy

Department store vs a street of shops (Core). The **distributed monolith** = a street of shops that can only open if **all** of them open, because they share one stockroom and one cash register: you got all the costs of separate shops and none of the freedom.

---

## How it works inside

### Request flow and its failure math

A page needs data from 5 services called **in sequence**, each 99.9% available and 20 ms:
```
Latency:       5 × 20 ms = 100 ms (+ network hops)
Availability:  0.999^5 ≈ 99.5%   (worse than any single service!)
```
Called **in parallel**: latency ≈ the slowest one (~20–30 ms), availability still ≈ 99.5% unless you **degrade gracefully** (show the page without recommendations if that service fails). Topic 23 does this math properly.

### Communication styles between services

| Style | How | Coupling | Use for |
|---|---|---|---|
| **Sync request/response** (REST/gRPC) | A calls B and waits | Temporal coupling: B down → A fails | Queries that need an answer now |
| **Async messaging / events** | A publishes "OrderPlaced"; B, C react later | Loose: B down → it catches up later | Side-effects, workflows, fan-out (topic 21) |
| **Shared database** | A and B read/write the same tables | ⚠️ Tight: schema changes break both | Avoid (distributed monolith smell) |

**Choreography** (services react to each other's events) vs **orchestration** (one coordinator tells each service what to do): topic 22.

### Drawing boundaries

Good signs:
- One team can own it end to end (code, data, on-call).
- It changes mostly on its own (features rarely touch 3 services).
- It owns its data; others ask via API/events.
- High cohesion inside, low coupling outside.

Bad signs (the **distributed monolith**):
- Services must be deployed together in a fixed order.
- Shared database tables across services.
- Chatty sync calls (a request fans out to 30 calls).
- A shared "common" library that forces everyone to upgrade together.
- "Nano-services" (one per entity/CRUD table).

### Migration: the strangler fig

Named after a plant that grows around a tree until it replaces it.

```
Step 1: [Proxy/gateway] → [Monolith (all features)]
Step 2: [Proxy] → /payments/* → [new Payments service + own DB]
                  everything else → [Monolith]
        (sync data: monolith publishes events / CDC (Change Data Capture: stream DB changes))
Step 3: repeat feature by feature; monolith shrinks; finally retire it
```

Pick the first service carefully: valuable, fairly independent, clear data ownership. Not the most tangled core.

### What microservices mean for DevOps

| Area | Monolith | Microservices |
|---|---|---|
| CI/CD | One pipeline | Templated pipelines per service, contract tests, independent versions (topic 32) |
| Deploy | One rollout | Many daily deploys; progressive delivery per service (topic 34) |
| Infra | A few VMs / one Deployment | K8s platform, namespaces per team, quotas (topic 38) |
| Observability | Logs + metrics | + tracing, service maps, per-service SLOs (topics 27–28) |
| Security | One perimeter | Service identity, mTLS, per-service IAM (topics 05, 40) |
| On-call | One rotation | Per-team ownership; clear service catalog |

---

## Options and trade-offs

| | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Team size sweet spot | 1–15 engineers | 10–50 | 50+ in many teams |
| Deploy independence | None | None (but cleaner code) | Full |
| Latency | In-process calls (ns) | In-process | Network calls (ms) |
| Consistency | ACID transactions | ACID transactions | Eventual, sagas |
| Scaling | Whole app | Whole app | Per service |
| Ops overhead | Low | Low | High (needs a platform) |
| Fault isolation | Low | Medium | High (if designed well) |
| Debugging | Easy (one stack trace) | Easy | Hard (tracing needed) |

**Senior default:** start with a **modular monolith**; extract services when a concrete pressure appears (a team needs independent deploys, a component has very different scaling or reliability needs, or a different tech is truly needed).

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Distributed monolith** | Coordinated deploys, cascading failures | Deploy dependency lists, shared tables | Re-draw boundaries, events, own data |
| **Cascading failure** | One slow service → callers' threads fill → everything slow | Latency spreads along the call graph in traces | Timeouts, circuit breakers, bulkheads (topic 24) |
| **Chatty calls** | High latency, high cross-AZ cost | Traces with dozens of spans per request | Aggregate APIs, batch, cache, rethink boundaries |
| **Inconsistent data across services** | Order paid but not shipped | Reconciliation jobs find mismatches | Outbox, sagas, idempotent consumers (topic 22) |
| **Version incompatibility** | Consumer breaks after provider deploy | 4xx/5xx after deploy | Backward-compatible APIs, contract tests (topic 06) |
| **Ownership gaps** | Nobody answers a page for a service | Service catalog with missing owners | Every service has an owner, runbook, SLO |
| **Platform overload** | Teams each build their own pipelines/monitoring | Duplication, drift | Golden paths (topic 39) |

**3 AM:** "Checkout latency 10×." With microservices, first open a **trace** of a slow checkout and find the span that's slow; then look at that service's dashboards. Without tracing, you're guessing across 20 services. That's why tracing is a must-have before splitting.

---

## Where you'll use it

- **Interviews:** "Would you use microservices here?" is asked constantly. Answer with team size, change patterns, and costs.
- **Platform work:** Kubernetes namespaces, CI templates, service catalogs exist mostly because of microservices.
- **Migrations:** strangler fig with an Ingress/gateway routing paths to new services.
- **Incident analysis:** cascading failures.

---

## Design exercise

**Prompt:** A 25-engineer startup has a monolith (Django + Postgres). Deploys take 40 minutes and happen twice a week; a slow PDF-generation feature causes timeouts for the whole site during month-end. The CTO wants "microservices everywhere". What do you propose?

<details>
<summary><b>Model answer</b></summary>

The two pains are **slow delivery** and **one heavy feature hurting everything**, not "we have too many teams".

1. **Fix delivery inside the monolith first:** parallelize tests, cache dependencies, build once and promote the same image (topic 33), deploy with rolling/canary on K8s. Goal: 10-minute pipeline, deploy on every merge.
2. **Make it a modular monolith:** enforce module boundaries (import rules), each module owns its tables.
3. **Extract the one thing with a real reason:** PDF generation goes **async**: the web request enqueues a job (SQS) and returns "we'll email you"; a separate worker deployment (same codebase or a small service) scales independently with KEDA on queue depth. This removes the month-end timeouts.
4. **Prerequisites before any further split:** tracing, per-service SLOs, CI templates, a service catalog.
5. Re-evaluate in 6–12 months: if teams grow to ~50 engineers and specific domains (payments, search) need independent deploys/scaling, extract them with the strangler fig pattern.

Trade-off: we accept a shared deploy for now in exchange for far lower operational load with 25 engineers.
</details>

---

## Quick recap

- Monolith = one deployable; modular monolith = one deployable with clean boundaries; microservices = independent services owning their data.
- Microservices trade organizational scale and independent deploys for distributed-system complexity.
- Sequential service calls multiply latency and failure probability.
- Boundaries by business capability and data ownership; avoid shared DBs and lock-step deploys (distributed monolith).
- Migrate with the strangler fig; build platform and tracing first.

### Interview questions

1. **Q: When are microservices the wrong choice?**
   A: When the team is small, the domain is still changing fast (boundaries unclear), there's no platform (CI/CD, observability, K8s) to support many services, or the real problem is something else (slow pipeline, one heavy feature). A modular monolith plus targeted async workers usually gives most benefits at a fraction of the cost.

2. **Q: How do you tell if you have a distributed monolith?**
   A: Services must deploy together or in a fixed order, they share database tables, a single user request makes long chains of synchronous calls, a shared library forces lock-step upgrades, and one service's outage takes down unrelated features. You pay the cost of distribution without independence.

3. **Q: Service A calls B, C and D synchronously for every request. How do you make this resilient?**
   A: Call them in parallel where possible, give each a timeout and a retry budget, use circuit breakers, and decide per dependency whether the page can degrade without it. Move side-effects to async events. Cache stable data from B/C/D. Measure the critical path with tracing and set SLOs per dependency.
