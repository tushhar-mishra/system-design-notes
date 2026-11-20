# 09 · Load Balancing

> **Level:** ★★ intermediate
>
> **You will learn:** what a **load balancer** does, **L4 vs L7** load balancing, the main **algorithms** (round robin, least connections, least outstanding requests, consistent hashing, weighted), **health checks** (and how a bad one causes an outage), **connection draining**, **sticky sessions** and why to avoid them, **slow start**, **cross-zone balancing**, **global vs regional** LBs, the AWS family (**ALB, NLB, GWLB**) with GCP/Azure equivalents, and how Kubernetes Services, Ingress and Gateway fit in.
>
> **You need:** topics 04–08.
>
> **Part 1 · Networking and the edge** · Previous: [08 · Cloud network](08-cloud-network-building-blocks.md) · Next: [10 · Proxies, gateways, discovery](10-proxies-gateways-service-discovery.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **load balancer** (LB) sits in front of many identical servers, gives clients **one address**, spreads requests across servers, and **stops sending traffic to dead ones** (health checks). This gives you **scale** (add servers) and **availability** (lose servers).

```
                 ┌─▶ app-1 (healthy)
Clients ─▶ [LB] ─┼─▶ app-2 (healthy)
                 └─✗ app-3 (failed health check → removed)
```

- **L4** (transport layer): balances **TCP/UDP connections** by IP and port. Fast, can't see HTTP.
- **L7** (application layer): understands **HTTP**: routes by path/host/header, terminates TLS, retries, per-request balancing.

**Picture it.** A supermarket with many checkout lanes and a person at the entrance pointing customers to the shortest queue, skipping any lane with a "closed" sign.

**The one thing to know.** **Health checks decide everything.** A health check that's too shallow sends traffic to broken servers; one that's too deep (checks the DB) can mark **all** servers unhealthy at once when the DB blips, causing a total outage.

**Try it now:** in any K8s cluster, `kubectl get endpointslices -n <ns>`. That's the list of healthy targets kube-proxy balances across.

**You've got it if you can:** choose L4 vs L7 for a case, pick an algorithm, design a health check, and explain connection draining.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**Load balancer** = a component that distributes incoming traffic across a pool of **targets** (also called backends, upstreams, members) and removes unhealthy ones.

The **OSI model** layers (a 7-layer way to describe networking) matter here only as names:
- **Layer 4** = transport (TCP/UDP): the LB sees IPs and ports.
- **Layer 7** = application (HTTP, gRPC): the LB sees URLs, headers, cookies.

Forms of load balancing:
- **Hardware/virtual appliances:** F5, Citrix (traditional data centres).
- **Software:** nginx, HAProxy, Envoy ([Linux 078](../../linux-mastery-rhel/notes/078-web-servers-apache-and-nginx.md)).
- **Cloud managed:** AWS ALB/NLB, GCP Cloud Load Balancing, Azure Load Balancer / Application Gateway.
- **DNS-based** (topic 07) and **anycast** global LBs.
- **Client-side:** the client picks a target from a list (gRPC, service meshes, Netflix-style).
- **Kernel-level:** kube-proxy (iptables/IPVS) or eBPF (Cilium) for Kubernetes Services.

---

## Why it exists

1. **Horizontal scale:** one server can't handle all traffic; ten behind an LB can.
2. **Availability:** failed servers are taken out automatically.
3. **Zero-downtime deploys:** drain old servers, add new ones (topic 34).
4. **One stable entry point:** clients don't need to know server IPs.
5. **Central place for TLS, WAF, routing, logging.**

---

## Analogy

The supermarket entrance (Core), extended:

- **Round robin** = send customers to lanes 1, 2, 3, 1, 2, 3...
- **Least connections** = send to the lane with the fewest people.
- **Health check** = checking the lane's light is on.
- **Connection draining** = when a cashier's shift ends, stop sending new customers but let current ones finish.
- **Sticky sessions** = a customer insists on the same cashier every visit. Fine until that cashier goes home.
- **L4 vs L7** = L4 directs by "which door did you come through"; L7 reads your shopping list and sends you to the specialist counter.

---

## How it works inside

### L4 vs L7 request flow

```
L4 (e.g. NLB):                                   L7 (e.g. ALB / Envoy):
Client ──TCP SYN──▶ LB                            Client ══TLS══▶ LB (terminates TLS)
LB picks target per CONNECTION (hash of 5-tuple)  LB reads HTTP: Host, path, headers
packets forwarded (often keeping client IP)       LB picks target per REQUEST
Client ◀──────────── target ──────────────▶        LB opens/reuses its own connection to target
LB never parses HTTP                              Can retry, rewrite, add X-Forwarded-For,
                                                  route /api → svc A, /img → svc B
```

### Algorithms

| Algorithm | How | Good for | Watch out |
|---|---|---|---|
| **Round robin** | Next target in turn | Similar requests, similar servers | Ignores load; slow requests pile up |
| **Weighted round robin** | Bigger servers get more | Mixed instance sizes, canaries (5% weight) | Static weights |
| **Least connections / least outstanding requests** | Target with fewest active requests | Requests of varying duration | Needs accurate counts; new targets get flooded (use slow start) |
| **Random with two choices (P2C)** | Pick 2 random, choose the less loaded | Large fleets, Envoy default-ish | — |
| **Hash (IP / header / consistent hashing)** | Same key → same target | Caches, sticky-ish routing, sharded backends | Hot keys overload one target; consistent hashing (topic 18) limits reshuffling |
| **Latency-based / EWMA** | Prefer faster targets | Heterogeneous performance | Can starve recovering targets |

### Health checks

```
Every 10 s, LB → GET /healthz on each target
  2 consecutive successes → healthy
  3 consecutive failures  → unhealthy (stop sending NEW requests)
```

- **Liveness-style (shallow):** "is my process up and can it serve HTTP?" Fast, no dependencies.
- **Readiness-style (deeper):** "am I ready to serve?" e.g. warmed up, config loaded, can reach *my own* critical local resources.
- ⚠️ **Don't check shared dependencies** (database, Redis, a downstream API) in the LB health check. If the DB has a 20-second blip, *every* target fails at the same time → the LB has zero healthy targets → 503 for everyone, even for requests that didn't need the DB. Handle dependency failures inside the app (degrade, topic 24).
- **Fail open:** many LBs (ALB, Envoy with panic threshold) route to **all** targets if all are unhealthy, assuming the health check itself is wrong. Know your LB's behaviour.

Kubernetes maps this to **readiness probes** (remove pod from Service endpoints) and **liveness probes** (restart the container) ([K8s 22](../../CKA-Certification-Course-2025/notes/22-health-probes.md)).

### Connection draining (deregistration delay)

```
Deploy starts → target marked "draining"
  LB stops sending NEW requests to it
  In-flight requests get up to N seconds (ALB default 300 s) to finish
  Then the target is removed and can be stopped
```

In Kubernetes, there's a race: the pod gets SIGTERM while kube-proxy/LB may still send it traffic for a few seconds. Fix: a `preStop` sleep of ~5–15 s, then graceful shutdown in the app ([K8s 19 · pod termination](../../CKA-Certification-Course-2025/notes/19-pod-lifecycle-restarts-and-termination.md), [Docker 12 · signals](../../Docker/notes/12-pid1-signals-and-shutdown.md)).

### Slow start

A new target gets traffic gradually (e.g. over 30–60 s) so cold caches / JIT-compiled runtimes (Java) aren't hit with full load at once.

### Cross-zone load balancing

LB nodes live in each AZ. With **cross-zone on**, each LB node spreads across targets in all AZs (even distribution). Off: each LB node only uses targets in its own AZ (uneven if AZs have different target counts, but avoids cross-AZ data charges). ALB: always on. NLB: off by default.

### Global vs regional

```
Global (anycast or DNS):  user → nearest healthy REGION
Regional (ALB/NLB):       → healthy AZ/target inside the region
In-cluster (kube-proxy, mesh, Ingress): → healthy POD
```

Three layers, each with its own health checks. Many outages come from one layer thinking things are healthy when another doesn't.

### AWS family and equivalents

| AWS | Layer | Use | GCP | Azure |
|---|---|---|---|---|
| **ALB** (Application LB) | L7 HTTP/HTTPS/gRPC, WebSockets | Web apps, path/host routing, WAF, OIDC auth | External/Internal Application LB | Application Gateway (regional), Front Door (global) |
| **NLB** (Network LB) | L4 TCP/UDP/TLS | Very high throughput, static IPs, non-HTTP, preserve client IP, PrivateLink | External/Internal passthrough Network LB, proxy Network LB | Azure Load Balancer |
| **GWLB** (Gateway LB) | L3 | Inserting firewalls/inspection appliances | — (Packet Mirroring/NGFW) | Gateway Load Balancer |
| **Global Accelerator** | Anycast L4 | Global static IPs, fast regional failover | Global external LB (anycast built in) | Front Door / cross-region LB |
| CloudFront | L7 CDN | Edge (topic 15) | Cloud CDN | Front Door / Azure CDN |

### Kubernetes

- **Service (ClusterIP):** L4 virtual IP; kube-proxy or eBPF picks a pod per connection ([K8s 32](../../CKA-Certification-Course-2025/notes/32-inside-a-service-kube-proxy.md)).
- **Service type LoadBalancer:** provisions a cloud NLB/LB ([K8s 31](../../CKA-Certification-Course-2025/notes/31-services-nodeport-loadbalancer-externalname-headless.md)).
- **Ingress / Gateway API:** L7 routing; on EKS the AWS Load Balancer Controller creates ALBs that target **pod IPs directly** (IP mode), skipping an extra NodePort hop ([K8s 70](../../CKA-Certification-Course-2025/notes/70-ingress-on-eks-path-routing.md), [K8s 72](../../CKA-Certification-Course-2025/notes/72-gateway-api-concepts.md)).

---

## Options and trade-offs

| Need | Choose | Why |
|---|---|---|
| HTTP routing by path/host, WAF, auth | **L7 (ALB / Envoy / nginx)** | Sees requests |
| gRPC / HTTP/2 between services | **L7 or client-side/mesh** | Balances per request, not per long connection |
| Millions of connections, static IPs, non-HTTP (MQTT, databases, game UDP) | **L4 (NLB)** | Fast, protocol-agnostic |
| Must preserve client source IP at the target | NLB (IP preserved) or L7 with `X-Forwarded-For` / PROXY protocol | |
| End-to-end TLS, LB must not decrypt | L4 passthrough | (topic 05) |
| Session state in server memory (legacy) | Sticky sessions (cookie) | ⚠️ Temporary; move sessions to Redis (topic 11) |
| Cache servers / sharded backends | Consistent hashing | Keeps key → server mapping stable |

**Sticky sessions trade-offs:** uneven load, lost sessions when a server dies, slow drains during deploys. Prefer **stateless** apps.

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Deep health check on shared DB** | DB blip → all targets unhealthy → 503 for everyone | `HealthyHostCount` drops to 0 across all AZs simultaneously | Shallow LB checks; dependency handling in app |
| **Health check too shallow** | Broken targets (e.g. bad config) still get traffic → 5xx | 5xx per target | Readiness that verifies the app can serve (local checks) |
| **No draining / no preStop** | 502s during every deploy | 5xx spikes aligned with deploys | Deregistration delay, preStop sleep, graceful shutdown |
| **Keep-alive timeout mismatch** | Sporadic 502 | ALB `HTTPCode_ELB_502_Count` | Backend idle timeout > LB idle timeout (topic 06) |
| **Uneven load with long-lived connections** | One pod hot, others idle | Per-pod RPS/CPU skew | L7/per-request balancing, max connection age |
| **New targets overwhelmed** | Latency spike right after scale-out | p99 on new instances | Slow start, warm-up |
| **LB itself not scaled** (self-managed or pre-warming) | Errors during sudden spikes | LB 5xx, surge queue | Managed LBs scale but need time; pre-warm for known events (contact provider), or NLB |
| **Retry storms at LB** | LB retries on 5xx multiply load on a struggling backend | Request count > client count | Limit retries, retry budgets (topic 24) |
| **All traffic to one AZ** | AZ overload | Per-AZ metrics | Cross-zone LB, balanced target counts per AZ |

**3 AM:** "503 at the ALB, `HealthyHostCount=0`." Ask: did *all* targets fail at once? Then it's a shared cause (health check depends on DB, or a bad deploy everywhere). Check the health check path manually from inside the VPC: `curl -v http://<pod-ip>:8080/healthz`. If the check is wrong, fix the check; if the app is broken, roll back.

---

## Where you'll use it

- Every web design: CDN → **ALB** → pods.
- **Kubernetes**: Services, Ingress/Gateway, AWS Load Balancer Controller.
- **Blue/green and canary**: weighted target groups (topic 34).
- **Databases**: read replicas behind a reader endpoint (an L4/DNS balancer) (topic 17).
- **Terraform**: `aws_lb`, `aws_lb_target_group` (health check settings, `deregistration_delay`, `slow_start`), `aws_lb_listener_rule`.
- **Ansible**: rolling deploys with `serial` that drain from the LB first ([Ansible 49](../../Ansible/notes/49-project-rolling-deploy.md)).

---

## Design exercise

**Prompt:** You run an HTTP API (REST) and a gRPC internal service on EKS. Requirements: TLS at the edge, path routing `/api/*` and `/admin/*` (admin restricted to the office IP range), zero 5xx during deploys, and the gRPC service must spread load evenly. Design the load balancing.

<details>
<summary><b>Model answer</b></summary>

- **Edge:** ALB (internet-facing) created by the AWS Load Balancer Controller from an Ingress/Gateway, ACM cert, WAF attached. Listener rules: `/api/*` → api target group; `/admin/*` → admin target group **with a condition on source IP** (office CIDR) plus app-level auth.
- **Targets:** IP mode (ALB → pod IPs directly). Health check `GET /readyz`, interval 10 s, healthy 2 / unhealthy 2, checking only local readiness (not DB).
- **Zero 5xx deploys:** rolling update with `maxUnavailable: 0`, `maxSurge: 25%`; readiness gates (the LB controller can inject a pod readiness gate so pods aren't "Ready" until the ALB sees them healthy); `preStop: sleep 15`; app handles SIGTERM gracefully; `deregistration_delay` 30 s (shorter than the default 300 s since requests are short); `terminationGracePeriodSeconds` 45.
- **gRPC:** don't rely on kube-proxy (L4 per connection). Options: a service mesh (Envoy/Linkerd per-request balancing) or gRPC client-side load balancing over a **headless Service** with periodic re-resolution, plus server-side `max_connection_age` so clients reconnect and spread.
- **Algorithm:** least outstanding requests on the ALB target group for the API (requests vary in duration).

Trade-offs: a mesh adds complexity; client-side balancing needs support in every client language.
</details>

---

## Quick recap

- LBs give one entry point, spread load, remove unhealthy targets, enable zero-downtime deploys.
- L4 balances connections (fast, protocol-agnostic); L7 balances requests (routing, TLS, retries).
- Choose algorithms by request shape: round robin, least outstanding, consistent hash.
- Health checks: shallow enough not to fail everything together, deep enough to catch broken targets.
- Draining + preStop + graceful shutdown = no 5xx on deploys. Avoid sticky sessions.

### Interview questions

1. **Q: Your health check calls the database. Is that a good idea?**
   A: Usually not for the LB check. A brief DB issue makes every target unhealthy simultaneously, and the LB has nothing to route to, so a partial degradation becomes a total outage. Keep LB/readiness checks focused on whether this instance can serve, and handle dependency failures in the app with timeouts, fallbacks and circuit breakers. Monitor the DB separately.

2. **Q: L4 or L7 for a TCP-based MQTT broker with a million IoT connections?**
   A: L4 (NLB). It handles millions of long-lived connections, passes TCP through without parsing, gives static IPs, and can preserve client IPs. L7 adds little since it's not HTTP, and costs more per connection.

3. **Q: Why do you get 502s during deploys and how do you fix them?**
   A: Pods are killed while the LB still sends them requests, or in-flight requests are cut. Fix: readiness probe fails/terminating pods are removed from endpoints, a `preStop` sleep covers propagation delay, the app finishes in-flight requests on SIGTERM, the LB deregistration delay matches request duration, and `maxUnavailable: 0` keeps capacity during rollout.
