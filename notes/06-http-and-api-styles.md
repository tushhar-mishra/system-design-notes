# 06 · HTTP and API Styles

> **Level:** ★ foundation
>
> **You will learn:** how **HTTP** works (methods, status codes, headers), the differences between **HTTP/1.1, HTTP/2 and HTTP/3**, the main **API styles** (REST, gRPC, GraphQL, WebSockets, webhooks, long polling) and when to choose each, **idempotency** and why it matters for retries, **pagination**, **API versioning**, and how status codes drive **load balancer, retry and alerting** behaviour.
>
> **You need:** topics 04–05. [Linux 070 · HTTP and curl](../../linux-mastery-rhel/notes/070-http-and-curl.md) covers curl hands-on.
>
> **Part 1 · Networking and the edge** · Previous: [05 · TLS](05-tls-and-https.md) · Next: [07 · DNS](07-dns-for-system-design.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** HTTP is request → response. The **method** says what you want (GET read, POST create, PUT replace, DELETE remove), the **status code** says what happened (2xx OK, 4xx your fault, 5xx our fault). API styles are different ways of using (or replacing) this.

```
REST       → resources + HTTP verbs, JSON. Default for public APIs.
gRPC       → typed functions over HTTP/2, protobuf. Fast service-to-service.
GraphQL    → client asks for exactly the fields it needs. Frontend-heavy apps.
WebSocket  → two-way, long-lived connection. Chat, live dashboards.
Webhook    → "we call you" when something happens. Integrations (GitHub → Jenkins).
```

**Picture it.** A restaurant order. **REST** = a fixed menu with standard orders. **GraphQL** = "I'd like exactly these 3 things from 3 dishes". **gRPC** = a kitchen-to-kitchen intercom with a strict order form. **WebSocket** = an open phone line to the kitchen. **Webhook** = "call me when my table is ready".

**The one thing to know.** **Idempotent** = doing it twice has the same effect as doing it once. GET, PUT, DELETE are idempotent; POST is not. Only retry idempotent requests automatically, or make POST idempotent with an **idempotency key**.

**Try it now:**

```bash
curl -sI https://example.com            # status line + headers
curl -s --http2 -o /dev/null -w "%{http_version}\n" https://www.google.com
```

**You've got it if you can:** pick an API style for a scenario with a reason, explain 502 vs 503 vs 504, and explain idempotency keys.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**HTTP** (HyperText Transfer Protocol) = the language of the web and of most APIs.

**A request:**
```
POST /v1/orders HTTP/1.1
Host: api.shop.example.com
Authorization: Bearer eyJ...
Content-Type: application/json
Idempotency-Key: 7f9c2b1e-...

{"product_id": 42, "qty": 1}
```

**A response:**
```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/orders/9001

{"id": 9001, "status": "pending"}
```

**Methods:**

| Method | Meaning | Safe (no change)? | Idempotent? |
|---|---|---|---|
| GET | Read | Yes | Yes |
| HEAD | Read headers only | Yes | Yes |
| POST | Create / do an action | No | **No** |
| PUT | Replace a resource fully | No | Yes |
| PATCH | Partial update | No | Not guaranteed |
| DELETE | Remove | No | Yes |

**Status codes you must know (for design and on-call):**

| Code | Meaning | Who's at fault | Retry? |
|---|---|---|---|
| 200 / 201 / 204 | OK / Created / No content | — | — |
| 301 / 302 / 307 / 308 | Redirect | — | Follow |
| 304 | Not modified (use your cached copy) | — | — |
| 400 | Bad request | Client | No |
| 401 | Not authenticated (who are you?) | Client | After login |
| 403 | Forbidden (I know you, but no) | Client | No |
| 404 | Not found | Client | No |
| 409 | Conflict (e.g. version mismatch) | Client | After re-reading |
| 429 | Too many requests (rate limited, topic 25) | Client | Yes, after `Retry-After` |
| 500 | Internal server error (app bug/exception) | Server | Maybe (if idempotent) |
| 502 | Bad gateway: proxy/LB got an invalid response or connection reset from upstream | Server (upstream) | Maybe |
| 503 | Service unavailable: overloaded, no healthy backends, maintenance | Server | Yes, with backoff |
| 504 | Gateway timeout: proxy/LB waited too long for upstream | Server (slow upstream) | Careful: the request may have succeeded |

> **Upstream** = the server behind a proxy (the proxy forwards requests "up" to it). **Downstream** = the caller side.

---

## Why it exists

You will choose API styles for every design, and on-call you will read status codes all day. The choice affects latency (HTTP/2 multiplexing, gRPC binary encoding), load balancing (long-lived connections), caching (GET is cacheable, POST isn't), and safety of retries (idempotency).

---

## Analogy

See Core. One more: **status codes are like a delivery driver's notes.** "Delivered" (200). "Wrong address" (404, your fault). "Recipient refused" (403). "Depot closed today" (503, try later). "Waited at the door 10 minutes, nobody answered" (504, and maybe the parcel was taken by a neighbour; you don't know if it arrived).

---

## How it works inside

### HTTP versions

```
HTTP/1.1                     HTTP/2                              HTTP/3
one request at a time        many streams on one TCP conn         many streams on QUIC (UDP)
per connection               (multiplexing), binary,             no TCP head-of-line blocking,
→ browsers open ~6 conns     header compression (HPACK)          0/1-RTT setup, connection
per host                     but TCP head-of-line blocking       migration (Wi-Fi → 4G)
```

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC over UDP |
| Requests per connection | One at a time (pipelining unused) | Many in parallel (streams) | Many, independent |
| Head-of-line blocking | Per connection | At TCP level | None between streams |
| Typical use | Legacy, simple clients | Default today; gRPC requires it | Browsers/CDNs to users on lossy mobile networks |

**Design consequence:** HTTP/2 and gRPC keep **one long-lived connection** with many requests. An L4 load balancer balances **connections**, not requests, so one client may pin all its traffic to one backend. → use L7 (request-level) balancing for gRPC (topic 09).

### API styles

**REST** (REpresentational State Transfer): resources as URLs (`/orders/9001`), HTTP methods as verbs, usually JSON.
```
GET    /v1/orders?status=open&limit=50
POST   /v1/orders
GET    /v1/orders/9001
PATCH  /v1/orders/9001
```

**gRPC**: you define services in a `.proto` file; code generators create client and server stubs in many languages. Binary **protobuf** (Protocol Buffers) encoding is smaller and faster than JSON. Supports streaming.
```proto
service Orders {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc WatchOrders(WatchRequest) returns (stream OrderEvent);
}
```

**GraphQL**: one endpoint (`POST /graphql`); the client sends a query naming exactly the fields it wants. Avoids over-fetching and multiple round trips for mobile/web frontends. Harder to cache at HTTP level and to rate limit (one query can be very expensive).

**WebSockets**: an HTTP request "upgrades" to a persistent two-way TCP channel. Server can push anytime. **SSE** (Server-Sent Events) is a simpler one-way server→client stream over plain HTTP.

**Long polling**: client sends a request; the server holds it open until there's news or a timeout, then the client immediately asks again. Simple fallback when WebSockets aren't possible.

**Webhooks**: the provider calls *your* URL when an event happens (GitHub → Jenkins on push, [Jenkins 46](../../Jenkins/notes/46-github-integration.md)). Must be authenticated (HMAC signature), idempotent (providers retry), and fast (queue the work, return 2xx).

### Idempotency keys

Problem: client sends `POST /payments`, the network drops the response. Did the payment happen? If the client retries, the customer may pay twice.

```
Client ── POST /payments  Idempotency-Key: abc-123 ──▶ API
API: key abc-123 seen before?
   no  → process, store (key → result) for 24 h, return result
   yes → return the stored result, do NOT charge again
```

The key is generated by the client (a UUID) once per logical operation and reused on every retry. Store keys in a DB with a unique constraint (or Redis with TTL). Topic 22 builds on this.

### Pagination

Never return unbounded lists.
- **Offset** (`?limit=50&offset=1000`): simple, but slow on big tables (DB scans and skips 1,000 rows) and items shift when new ones are inserted.
- **Cursor / keyset** (`?limit=50&after=order_9001`): the server uses `WHERE id > 9001 ORDER BY id LIMIT 50`, which is fast with an index and stable. Preferred at scale.

### Versioning

- URL (`/v1/...`): obvious, easy to route at the gateway. Most common.
- Header (`Accept: application/vnd.shop.v2+json`): cleaner URLs, harder to debug.
- Rule: **additive changes are backward compatible** (new optional fields). Removing/renaming fields needs a new version and a deprecation period. Same idea as zero-downtime DB changes (topic 35).

### Caching headers (preview of topics 14–15)

`Cache-Control: public, max-age=300` lets browsers and CDNs cache for 5 minutes. `ETag` + `If-None-Match` lets the client ask "has it changed?" and get a cheap `304`.

---

## Options and trade-offs

| Style | Best for | Pros | Cons |
|---|---|---|---|
| **REST/JSON** | Public APIs, CRUD, broad compatibility | Simple, cacheable GETs, human-readable, every tool supports it | Over/under-fetching, no strict schema unless OpenAPI enforced |
| **gRPC** | Internal service-to-service, low latency, streaming | Fast binary, strict contracts, codegen, deadlines built in | Not browser-native (needs gRPC-Web), harder to debug with curl, needs L7 LB |
| **GraphQL** | Frontends aggregating many backends | Client picks fields, one round trip | Caching, rate limiting and query cost control are harder; N+1 query risk |
| **WebSocket / SSE** | Real-time push (chat, live logs, dashboards) | Instant updates | Stateful long-lived connections: LB idle timeouts, harder scaling and deploys |
| **Long polling** | Simple real-time with any infra | Works everywhere | Wasteful, more latency |
| **Webhooks** | Event notifications to third parties | No polling | Delivery failures, security (verify signatures), retries/duplicates |
| **Async via queue** (topic 21) | Work that can happen later | Absorbs spikes, decouples | Caller doesn't get the result immediately |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Retrying non-idempotent POST** | Duplicate orders / payments | Duplicate-record checks, customer complaints | Idempotency keys; only auto-retry idempotent methods |
| **504 from LB but backend finished** | Client retries, double work | Correlate LB 504s with backend logs (request IDs) | Timeouts aligned (topic 24); idempotency |
| **LB idle timeout < keep-alive** | Random 502s on reused connections | 502 bursts with "connection reset" | Backend keep-alive timeout **greater than** LB idle timeout (e.g. ALB 60 s → app 65 s) |
| **gRPC behind L4 LB** | One backend at 100% CPU, others idle | Per-pod request rate skew | L7 LB / mesh, or client-side balancing |
| **WebSocket connections all on old pods after deploy** | Uneven load, slow drains | Connection count per pod | Graceful drain with reconnect + jitter on the client |
| **Unbounded list endpoint** | Slow queries, memory spikes on large accounts | p99 latency for one tenant | Mandatory pagination with max limit |
| **Breaking API change** | Clients fail after deploy | 4xx spike from specific clients | Versioning, contract tests, additive changes only |
| **Webhook storm** (provider retries) | Spike of duplicate events | Rate of identical delivery IDs | Ack fast, process async, dedupe by delivery ID |

**3 AM:** "502s spiking at the load balancer." 502 means the LB **got a bad answer or a reset** from upstream. Usually: pods crashing/restarting, pods killed mid-request during a deploy (no graceful shutdown, topic 34), or keep-alive timeout mismatch. 504 means **upstream too slow**: look at backend latency and dependencies (DB). 503: no healthy targets or overload.

---

## Where you'll use it

- **Health check endpoints** (`/healthz`, `/readyz`) return 200/503 to LBs and K8s probes ([K8s 22](../../CKA-Certification-Course-2025/notes/22-health-probes.md)).
- **Alerting** on 5xx rate, not 4xx (4xx is usually the client's fault, but a 4xx spike after a deploy can mean a breaking change).
- **Webhooks** from GitHub/GitLab into CI ([GHA 09 · triggers](../../GitHub-Actions-Basics-To-Production/notes/09-workflow-triggers.md)).
- **Choosing gRPC vs REST** for internal services in a microservice design (topic 13).

---

## Design exercise

**Prompt:** Design the API between a mobile app and your backend for "place order" and "track delivery live on a map". Mobile networks are flaky.

<details>
<summary><b>Model answer</b></summary>

- **Place order:** `POST /v1/orders` (REST/JSON over HTTPS; HTTP/3 at the CDN edge helps on lossy mobile networks). The app generates an **Idempotency-Key** per checkout and retries with the same key on timeout. The server stores key → result in the DB (unique index) for 24 h. Response `201` with order ID. Payment processing goes through a queue (topic 21), and the app polls `GET /v1/orders/{id}` or receives a push notification.
- **Live tracking:** a **WebSocket** (or SSE) connection to a tracking gateway that pushes driver location every few seconds. On reconnect, the client sends the last event ID to resume. Fallback: polling `GET /v1/deliveries/{id}/location` every 10 s.
- **Backend:** internal services talk **gRPC** with deadlines.
- Trade-offs: WebSockets mean long-lived connections, so set LB idle timeouts (e.g. 300 s + app-level pings every 30 s), drain gracefully on deploys, and cap connections per pod. Polling is simpler but costs more requests and battery.
</details>

---

## Quick recap

- HTTP = method + path + headers + body → status + headers + body. 4xx = client, 5xx = server.
- 502 bad upstream answer, 503 unavailable/overloaded, 504 upstream too slow (the work may have happened).
- HTTP/2 multiplexes on one TCP connection; HTTP/3 uses QUIC/UDP; long-lived connections need L7 balancing.
- REST for public, gRPC for internal, GraphQL for flexible frontends, WebSocket/SSE for push, webhooks for event callbacks.
- Idempotency keys make retries safe; prefer cursor pagination; version and change APIs additively.

### Interview questions

1. **Q: A client timed out calling `POST /transfer`. Should it retry?**
   A: Only if the operation is idempotent. A timeout doesn't tell you whether it succeeded. The safe design is an idempotency key: the client sends the same key on retry, and the server returns the original result instead of transferring again.

2. **Q: Why might gRPC traffic overload one pod while others are idle?**
   A: gRPC uses HTTP/2, which multiplexes all requests over one long-lived connection. An L4 load balancer (or kube-proxy) balances connections, not requests, so each client sticks to one pod. Use L7 load balancing (Envoy, a mesh, or an L7 gateway) or client-side load balancing with periodic reconnection.

3. **Q: How do you change an API without breaking clients?**
   A: Make changes additive (new optional fields, new endpoints), never repurpose fields, version breaking changes (`/v2`) and run both versions during a deprecation period, track usage of the old version, and use contract tests in CI so a provider change that breaks a consumer fails the build.
