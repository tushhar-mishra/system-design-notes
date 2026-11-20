# 10 · Reverse Proxies, API Gateways and Service Discovery

> **Level:** ★★ intermediate
>
> **You will learn:** **forward vs reverse proxy**, what a reverse proxy does (TLS, routing, buffering, compression, caching), what an **API gateway** adds (auth, rate limits, API keys, request transformation), how services **find each other** (**service discovery**: DNS, registries, client-side vs server-side), how **Kubernetes Services** and **EndpointSlices** implement discovery, and a one-page view of the **service mesh** (sidecar vs ambient) with its costs.
>
> **You need:** topics 05–09.
>
> **Part 1 · Networking and the edge** · Previous: [09 · Load balancing](09-load-balancing.md) · Next: [11 · Scalability basics](11-scalability-basics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.**
- A **reverse proxy** sits in front of servers and handles the boring network work for them (TLS, routing, compression, buffering slow clients).
- An **API gateway** is a reverse proxy that also enforces **API rules**: who you are (auth), how much you may call (rate limits), which version you get.
- **Service discovery** answers "where is service X right now?" because in Kubernetes and autoscaling groups, IPs change all the time.
- A **service mesh** puts a proxy next to (or under) every service so service-to-service traffic gets mTLS, retries, metrics and routing without code changes.

```
Internet → [API gateway: auth, rate limit, /v1 routing] → [service A] ──(discovery: "where is B?")──▶ [service B]
                                                              (mesh proxies on both sides: mTLS, retries, metrics)
```

**Picture it.** A hotel. The **front desk** (reverse proxy) greets guests and sends them to the right room. The **concierge with a guest list** (API gateway) checks IDs and limits room service orders. The **internal phone directory** (service discovery) tells staff the current extension of each department, which changes when departments move.

**The one thing to know.** Put **cross-cutting concerns** (TLS, auth, rate limits, retries, telemetry) in shared infrastructure once, instead of re-implementing them in every service. But every layer you add is another hop, another config, another thing that can fail.

**Try it now:** `kubectl get endpointslices -A | head`: the live "phone directory" of the cluster.

**You've got it if you can:** say what goes in the gateway vs the app, compare client-side and server-side discovery, and argue for or against a service mesh.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Component | Sits | Acts for | Examples |
|---|---|---|---|
| **Forward proxy** | Near clients | Clients (outbound) | Corporate web proxy, Squid, egress proxy for allow-listing outbound domains |
| **Reverse proxy** | Near servers | Servers (inbound) | nginx, HAProxy, Envoy, Traefik, Caddy |
| **API gateway** | At the API front door | API owners | AWS API Gateway, Kong, Apigee, Envoy Gateway, Azure API Management, GCP API Gateway/Apigee |
| **Ingress controller / Gateway API implementation** | Edge of a K8s cluster | Cluster services | ingress-nginx, AWS LB Controller, Envoy Gateway, Istio, Cilium |
| **Service registry** | Anywhere | Everyone looking up services | Kubernetes API + CoreDNS, Consul, Eureka, AWS Cloud Map |
| **Service mesh** | Next to every service | Service-to-service traffic | Istio (sidecar or ambient), Linkerd, Cilium service mesh |

---

## Why it exists

Without a reverse proxy/gateway, **every service** must implement TLS, auth, rate limiting, logging, CORS, compression, request size limits... in each language, consistently. Without discovery, configs hard-code IPs that break when pods move. Without a mesh (at large scale), every team implements retries/timeouts/mTLS differently and inconsistently.

---

## Analogy

The hotel (Core). Extensions:
- **Forward proxy** = the hotel's own outgoing mail room: staff's letters go out through it, and it can refuse to post to unknown addresses (egress control).
- **Buffering** = the front desk lets a slow-talking guest finish their whole request before calling the busy chef, so the chef isn't kept waiting.
- **Mesh sidecar** = every department gets its own personal assistant who handles calls, security checks and note-taking.

---

## How it works inside

### Reverse proxy duties

```
Client ══TLS══▶ [Reverse proxy]
                 1. terminate TLS (topic 05)
                 2. read the whole request (buffer slow clients → protects app threads)
                 3. route by host/path → upstream pool
                 4. add headers: X-Forwarded-For, X-Request-ID
                 5. enforce limits: body size, timeouts, connection limits
                 6. compress (gzip/brotli), maybe cache (topic 14)
                 7. load-balance across upstreams with health checks (topic 09)
                 8. log the request with latency and status
                ───▶ [app]
```

Request IDs: generate one at the edge and pass it to every downstream call and log line. It's how you trace one request across services during incidents (topic 27).

### API gateway additions

```
Request → [Gateway]
   ├─ AuthN: validate JWT / API key / mTLS (who are you?)
   ├─ AuthZ: scopes, tenant checks (may you call this?)
   ├─ Rate limit / quota per key/tenant (topic 25)
   ├─ Request validation (schema), size limits
   ├─ Routing + versioning (/v1 → old service, /v2 → new)
   ├─ Transformations (headers, protocol: REST → gRPC)
   ├─ Caching of GET responses
   └─ Usage analytics, logs
   → backend services (which can now trust the authenticated identity header)
```

> **JWT** (JSON Web Token) = a signed token carrying claims like user ID and expiry; the gateway verifies the signature without a DB lookup.

**What NOT to put in the gateway:** business logic. A gateway full of per-endpoint scripts becomes a fragile shared monolith that every team must change.

### Service discovery

**Problem:** service B runs 12 pods whose IPs change on every deploy and scale event. How does A find them?

**Server-side discovery** (A calls a stable name/VIP; something else picks the instance):
```
A ──▶ "orders.prod.svc.cluster.local" ──DNS──▶ ClusterIP 10.96.0.15
A ──▶ 10.96.0.15 ──kube-proxy/eBPF rule──▶ one of the ready pod IPs
```
- Kubernetes **Service** = stable name + virtual IP. **EndpointSlices** = the current list of **ready** pod IPs, updated by the control plane when pods become ready/unready.
- Cloud LBs (internal ALB/NLB) are also server-side discovery.

**Client-side discovery** (A gets the list and picks itself):
```
A ──▶ registry (DNS SRV / headless Service / Consul): "give me all B instances"
A ◀── [10.0.1.4, 10.0.2.9, 10.0.3.7]
A picks one (round robin, least-loaded) and connects directly
```
- Kubernetes **headless Service** (`clusterIP: None`) returns all pod IPs in DNS.
- gRPC clients, mesh proxies (Envoy gets endpoint lists via **xDS**, its discovery API).

**Registration:** in Kubernetes it's automatic (the kubelet reports pod readiness). Outside K8s: instances register in Consul/Cloud Map with health checks, or the platform (ASG + target group) does it.

### Service mesh in one page

```
Sidecar model:                                 Ambient / node-proxy model:
[app A]⇄[proxy] ═mTLS═ [proxy]⇄[app B]        [app A] → [node proxy (ztunnel)] ═mTLS═ [node proxy] → [app B]
 every pod has its own Envoy                   L4 mTLS per node; optional L7 "waypoint" proxies per service
                     ▲                                          ▲
              [control plane: config, certs, policies]  (Istio istiod, Linkerd control plane)
```

What a mesh gives: **mTLS + workload identity** (topic 05), per-request **load balancing** (fixes gRPC, topic 09), **retries/timeouts/circuit breaking** as config (topic 24), uniform **golden-signal metrics** and trace propagation (topic 27), **traffic splitting** for canaries (topic 34), authorization policies by identity.

What it costs: extra hops (~0.5–3 ms per hop with sidecars), CPU/memory per sidecar (can be 10–20% of a cluster's resources at scale), upgrade complexity, a new control plane that can break traffic, and a steep learning curve. ([K8s 74 · service mesh essentials](../../CKA-Certification-Course-2025/notes/74-service-mesh-essentials.md).)

---

## Options and trade-offs

### Gateway choices

| Option | Pros | Cons | Choose when |
|---|---|---|---|
| Cloud-managed gateway (AWS API Gateway, Apigee, Azure APIM) | No servers, auth/keys/quotas built in, scales | Per-request cost at high volume, vendor-specific, payload/timeout limits (e.g. 29 s default on AWS REST APIs, configurable) | Public APIs, moderate volume, serverless backends |
| Self-hosted (Kong, Envoy Gateway, nginx) on K8s | Cheap at volume, flexible, portable | You run, scale and patch it | High volume, K8s-native platforms |
| ALB + app-level auth | Simple | Each service does auth; less consistent | Few services, early stage |

### Discovery choices

| | Server-side (VIP/LB) | Client-side (registry + smart client) |
|---|---|---|
| Client complexity | Low | High (library per language) |
| Extra hop | Sometimes (LB) / none (kube-proxy) | None |
| Load balancing quality | Per connection (L4) | Per request, smarter algorithms |
| Failure modes | LB/VIP issues | Stale lists in clients |
| Examples | K8s ClusterIP, internal ALB | gRPC + headless svc, Envoy/mesh, Eureka |

### Mesh or not

| Situation | Recommendation |
|---|---|
| < ~10–20 services, one team | No mesh. Use Ingress/Gateway + good client libraries + NetworkPolicies. |
| Strict mTLS/zero-trust compliance across many services | Mesh (Linkerd for simplicity, Istio ambient for lower overhead and features). |
| Heavy gRPC with uneven load | Mesh or client-side LB. |
| Team without capacity to run another control plane | Not yet; it will page you. |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Gateway as SPOF** | All APIs down together | Gateway 5xx, synthetic checks | Multiple replicas across AZs, PDBs, autoscale, managed service |
| **Bad gateway config push** | Global outage from one change (common in big outages) | Error spike right after config change | Config validation in CI, staged rollout of gateway config, fast rollback |
| **Gateway timeout < backend time** | 504s while backends still working | 504 rate | Align timeouts from edge inward (outer ≥ inner + margin) (topic 24) |
| **Stale discovery** | Calls to dead IPs → connection refused | Errors after scale-down/deploy | Readiness + EndpointSlice propagation, client re-resolution, retries on connect errors |
| **DNS caching in clients** | Traffic to old pods after rollout | Old pod IPs still receiving connections | Low TTL for headless services, connection max age |
| **Mesh control plane down** | Usually: existing config keeps working, but new pods get no config/certs; when certs expire, traffic fails | Control plane health, cert expiry | HA control plane, monitor cert issuance, rehearse upgrades |
| **Sidecar not ready at startup / killed before app** | App's first calls fail; Jobs never finish | Startup errors | Native sidecar containers (K8s 1.29+ restartable init containers), `holdApplicationUntilProxyStarts` |
| **Proxy buffering limits** | 413 (too large) or slow uploads | 413 counts | Set body size limits per route deliberately |

**3 AM:** "Everything behind the gateway returns 401." A shared auth component failed: expired JWKS signing keys, an IdP (identity provider) outage, or a config change. Check the gateway's auth plugin logs and the IdP status. Mitigation: roll back the config; long-term, cache signing keys and alert on IdP errors.

---

## Where you'll use it

- **Kubernetes edge:** Ingress/Gateway API ([K8s 69](../../CKA-Certification-Course-2025/notes/69-ingress-concepts.md), [K8s 73](../../CKA-Certification-Course-2025/notes/73-gateway-api-hands-on.md)).
- **Platform design:** offering teams a standard gateway with auth and rate limits (topic 39).
- **Egress control:** forward proxy or egress gateway allow-listing external domains (topic 40).
- **Rate limiter case study** (topic 55).
- **nginx as reverse proxy** in front of containers ([Docker 39a](../../Docker/notes/39a-tls-and-reverse-proxies.md)).

---

## Design exercise

**Prompt:** You expose 15 microservices as a public API for partners. Partners authenticate with API keys and have different quotas (free: 10 req/s, premium: 200 req/s). Internally, services are on EKS. Design the front door and service-to-service discovery.

<details>
<summary><b>Model answer</b></summary>

- **Front door:** CloudFront (DDoS absorption, TLS at edge) → **API gateway**. Choice depends on volume: at, say, 5,000 req/s sustained, a managed per-request-priced gateway gets expensive (estimate: 5,000 × 86,400 × 30 ≈ 13 B requests/month), so a **self-hosted gateway (Kong or Envoy Gateway) on EKS** behind an NLB or ALB is cheaper. At low volume, managed API Gateway with usage plans is simplest.
- **Gateway duties:** API-key validation (keys stored hashed; key → partner → plan), **per-key rate limiting** backed by Redis for a cluster-wide count (topic 25), request size limits, `/v1`/`/v2` routing, request IDs, access logs with partner ID.
- **Behind the gateway:** services trust a signed identity header (or a gateway-issued internal JWT) and do their own fine-grained authorization.
- **Discovery:** Kubernetes Services + CoreDNS for REST services; gRPC services use headless Services with client-side balancing, or adopt a mesh later if mTLS becomes a requirement.
- **Resilience:** gateway replicas across 3 AZs, HPA, PodDisruptionBudget; gateway config in Git, validated in CI and rolled out progressively (canary gateway pods).
- Trade-offs: self-hosting means we own scaling and patching of the most critical component; mitigated by running it as a platform product with SLOs (topic 28).
</details>

---

## Quick recap

- Reverse proxy: TLS, routing, buffering, headers, limits, logging. API gateway: + auth, quotas, versioning, analytics.
- Keep business logic out of gateways; protect gateway config changes like code deploys.
- Service discovery: server-side (K8s Service VIP, LB) vs client-side (registry + smart client).
- Kubernetes: Service + EndpointSlices + CoreDNS = built-in discovery of ready pods.
- Service mesh = mTLS, per-request LB, retries, telemetry as infrastructure, at a real cost in complexity and resources.

### Interview questions

1. **Q: What belongs in an API gateway and what doesn't?**
   A: Cross-cutting edge concerns: authentication, coarse authorization, rate limits/quotas, request validation and size limits, routing/versioning, TLS, logging. Not business logic or per-endpoint data transformations; those make the gateway a shared bottleneck that every team needs to change, and a risky single point of failure.

2. **Q: How does service discovery work in Kubernetes?**
   A: A Service gives a stable DNS name and virtual IP. The control plane keeps EndpointSlices listing ready pod IPs (based on readiness probes). CoreDNS resolves the name to the ClusterIP, and kube-proxy (iptables/IPVS) or eBPF rewrites the connection to one ready pod. Headless Services return pod IPs directly for client-side balancing.

3. **Q: Would you introduce a service mesh for a 12-service application?**
   A: Only with a concrete need: mandatory mTLS/zero-trust, gRPC load-balancing problems, or traffic-splitting for canaries that we can't get otherwise. Otherwise the cost (control plane to operate, upgrades, latency, sidecar resources) outweighs the benefit, and Gateway API + NetworkPolicies + good client libraries cover most needs. If we do adopt one, I'd pick the simplest that meets the need and roll out namespace by namespace.
