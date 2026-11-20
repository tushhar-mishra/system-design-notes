# 04 · Networking Essentials for Design

> **Level:** ★ foundation
>
> **You will learn:** the networking you need to reason about designs: **IP addresses, ports, TCP vs UDP**, what a **connection** costs, **latency vs bandwidth vs throughput**, **RTT** and why distance matters, **connection reuse (keep-alive) and pooling**, **NAT and ephemeral ports**, **MTU**, and the networking failures that page you at 3 AM (port exhaustion, SYN backlog, packet loss).
>
> **You need:** topics 02–03. Your [Linux 058 · How networks work](../../linux-mastery-rhel/notes/058-how-networks-work.md) and [Linux 095 · TCP/IP deep dive](../../linux-mastery-rhel/notes/095-tcp-ip-deep-dive.md) go deeper on the packet level.
>
> **Part 1 · Networking and the edge** · Previous: [03 · Estimation](03-back-of-the-envelope-estimation.md) · Next: [05 · TLS and HTTPS](05-tls-and-https.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every arrow in a design diagram is a network call. Each one costs **round trips**, can **fail**, and has **limits** (connections, ports, bandwidth). Good designs minimize round trips, reuse connections, and expect the network to break.

```
TCP connection setup  = 1 RTT (SYN, SYN-ACK, ACK)
+ TLS 1.3 handshake   = 1 RTT
+ HTTP request        = 1 RTT
= 3 RTTs before the first byte on a NEW connection
   (same AZ: ~1 ms total; Mumbai→US: ~600 ms!)
Reuse the connection → only 1 RTT per request after that.
```

**Picture it.** A phone call. Dialling and saying "hello, can you hear me?" (handshake) takes time before you say anything useful. If you need to ask 20 questions, you don't hang up and redial after each one: you keep the line open (keep-alive).

**The one thing to know.** **Latency** (how long one trip takes) and **bandwidth** (how much fits through per second) are different. A wider pipe does not make a far-away server closer.

**Try it now:**

```bash
ping -c 3 8.8.8.8                       # RTT to a nearby Google server
curl -o /dev/null -s -w "connect=%{time_connect}s tls=%{time_appconnect}s\n" https://example.com
ss -s                                    # socket summary on your Linux box
```

**You've got it if you can:** explain the 3 RTTs before the first byte, the difference between TCP and UDP, and why connection pools exist.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **IP address** = the address of a machine (or network interface) on a network. IPv4 looks like `10.0.1.25`; IPv6 like `2001:db8::25`.
- **Private IP ranges** (RFC 1918) = addresses for internal networks, not reachable from the internet: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. The `/8` is **CIDR** notation (Classless Inter-Domain Routing): how many leading bits are the network part. `/16` = 65,536 addresses, `/24` = 256. ([Linux 059 · subnetting](../../linux-mastery-rhel/notes/059-ip-addressing-and-subnetting.md).)
- **Port** = a number (0–65535) that picks which program on the machine gets the traffic. HTTPS is 443, SSH 22, Postgres 5432.
- **Socket / connection** = one conversation, identified by the **5-tuple**: protocol, source IP, source port, destination IP, destination port.
- **TCP** (Transmission Control Protocol) = reliable, ordered delivery. It sets up a connection, numbers every byte, re-sends lost packets, and slows down when the network is congested.
- **UDP** (User Datagram Protocol) = fire-and-forget packets. No connection, no retries, no ordering. Fast and simple.
- **RTT** (Round-Trip Time) = time for a packet to go there and a reply to come back.
- **Latency** = delay for one operation. **Bandwidth** = maximum data per second a link can carry. **Throughput** = data per second you actually achieve.

---

## Why it exists (why you must know this for design)

Design decisions that depend on networking:

1. **Where to put things.** Putting the app in Mumbai and the DB in Virginia adds ~200 ms to **every** query.
2. **How many calls a request makes.** 10 sequential service calls × 2 ms = 20 ms in one AZ, but × 200 ms = 2 s across regions.
3. **Connection limits.** A database allows maybe a few hundred to a few thousand connections. 500 pods × 20 connections each = 10,000 → the DB falls over.
4. **Protocol choice.** Video calls and DNS use UDP because a late packet is useless; payments use TCP because lost bytes are unacceptable.
5. **Failure.** Networks drop packets, partition, and get slow. Your design must handle it (topic 24).

---

## Analogy

| Network idea | Real life |
|---|---|
| IP address | Building's street address |
| Port | Flat number inside the building |
| TCP | Registered post with tracking: signed for, re-sent if lost, arrives in order |
| UDP | Throwing flyers into letterboxes: fast, no guarantee |
| RTT | Time to send a letter and get the reply |
| Bandwidth | Width of the motorway (lanes) |
| Latency | Length of the motorway |
| Connection reuse | Keeping a phone call open instead of redialling |

A 10-lane motorway (bandwidth) to another country doesn't make the trip shorter (latency).

---

## How it works inside

### TCP connection lifecycle

```
Client                                   Server
  │ ── SYN ─────────────────────────────▶ │   1st half RTT
  │ ◀──────────────────────── SYN-ACK ── │   (server: half-open, in SYN backlog)
  │ ── ACK + data ──────────────────────▶ │   connection ESTABLISHED (1 RTT)
  │        ... requests / responses ...   │
  │ ── FIN ─────────────────────────────▶ │   close
  │ ◀───────────────────────────── ACK ── │
  │ ◀───────────────────────────── FIN ── │
  │ ── ACK ─────────────────────────────▶ │
  │  (client side waits in TIME_WAIT ~60 s on Linux before the port is reusable)
```

Key mechanics:

- **Three-way handshake** = 1 RTT before any data.
- **Slow start**: a new TCP connection starts by sending only a little data (the **congestion window**), then doubles it each RTT while no packets are lost. So a **new** connection is slow for big transfers; a **warm** reused one is fast.
- **Retransmission**: a lost packet is re-sent after a timeout, which can add hundreds of ms. 1% packet loss can halve throughput.
- **Head-of-line blocking**: TCP delivers bytes in order, so one lost packet blocks everything after it on that connection until it's re-sent. (HTTP/3 over QUIC fixes this, topic 06.)
- **TIME_WAIT**: after closing, the side that closed first keeps the 5-tuple reserved for a while so late packets don't confuse a new connection. Many short connections → many TIME_WAIT sockets.

### UDP

```
Client ── datagram ──▶ Server      (no handshake, no ACK, no order)
```

Used by DNS (topic 07), video/voice, gaming, metrics (StatsD), and **QUIC** (the base of HTTP/3, which rebuilds reliability on top of UDP in user space).

### Latency vs bandwidth: worked example

Download a 10 MB file from a server 200 ms RTT away over a 1 Gbps link.
- Pure transfer time at full speed: 10 MB × 8 = 80 Mb ÷ 1,000 Mb/s = 0.08 s.
- But a new connection needs handshakes (~2–3 RTTs = 400–600 ms) and slow start ramps up over several RTTs (another few hundred ms).
- Real time: ~1 s. **Latency dominates**, not bandwidth.

For **many small requests** (APIs), latency matters most. For **big bulk transfers** (backups, images), bandwidth matters most.

**Bandwidth-delay product** = bandwidth × RTT = how much data is "in flight" on the wire. 1 Gbps × 0.2 s = 25 MB. TCP buffers must be at least this big to fill the pipe. That's why cross-region backups can be slow even on fat links unless tuned ([Linux 055 · sysctl](../../linux-mastery-rhel/notes/055-kernel-modules-and-sysctl.md) `net.core.rmem_max` etc.) or parallelised.

### Connection reuse: keep-alive and pools

```
Without reuse:  [handshake][req][close] [handshake][req][close] [handshake][req][close]
With keep-alive: [handshake][req][req][req][req][req]...
```

- **HTTP keep-alive** keeps a TCP connection open for more requests.
- A **connection pool** is a set of already-open connections that an app borrows and returns. Every serious app uses one for its database and HTTP clients.
- **Server side:** a proxy like **PgBouncer** pools *database* connections so 500 pods share, say, 100 real DB connections.

### NAT and ephemeral ports

**NAT** (Network Address Translation) rewrites addresses so many private machines share one public IP (your home router; a cloud **NAT gateway**, topic 08).

Each outgoing connection needs a **source port**. The OS picks them from the **ephemeral port range** (Linux default `32768–60999` → ~28,000 ports). A connection is unique per 5-tuple, so **one source IP talking to one destination IP:port can have at most ~28k concurrent connections** (plus TIME_WAIT ones still holding ports).

→ **Port exhaustion**: a busy service behind one NAT IP, opening new connections per request to one API, runs out of ports. Symptom: random connection timeouts. Fixes: reuse connections, add NAT IPs, widen the range (`net.ipv4.ip_local_port_range`).

### MTU

**MTU** (Maximum Transmission Unit) = biggest packet a link carries, usually 1,500 bytes on Ethernet (9,001 "jumbo frames" inside AWS VPCs). Tunnels (VPNs, overlays like VXLAN in Kubernetes CNIs) add headers, so the usable size shrinks. If something in the path blocks the ICMP "too big" messages, large packets silently vanish while small ones work: the classic **"SSH works but `git clone` hangs"** symptom. ([K8s 29 · network model and CNI](../../CKA-Certification-Course-2025/notes/29-network-model-and-cni.md).)

---

## Options and trade-offs

### TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Reliability | Guaranteed, ordered | None (app must handle) |
| Setup cost | 1 RTT handshake | None |
| Head-of-line blocking | Yes | No |
| Use it for | APIs, databases, file transfer, anything that must arrive | DNS, real-time media, games, telemetry, QUIC/HTTP/3 |

### Connection strategies

| Strategy | Pros | Cons | Choose when |
|---|---|---|---|
| New connection per request | Simple, no stale connections | Handshake cost, TIME_WAIT, port exhaustion | Very low volume only |
| Keep-alive / client pool | Fast, fewer handshakes | Stale connections after server restarts; uneven LB spread | Default for every service |
| Server-side pooler (PgBouncer, RDS Proxy) | Protects DB from thousands of clients | Another component; some session features limited | Many pods → one DB |
| Multiplexed protocol (HTTP/2, gRPC) | Many requests on one connection | Long-lived connections defeat L4 load balancing | Service-to-service calls (topic 06, 09) |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Ephemeral port exhaustion** | Intermittent `connect: cannot assign requested address`, timeouts at high load | `ss -s` shows tens of thousands in TIME_WAIT; NAT gateway `ErrorPortAllocation` metric | Connection pooling/keep-alive; more NAT IPs; VPC endpoints for AWS services (topic 08) |
| **Too many DB connections** | `FATAL: too many connections`, DB memory spikes | DB connection count metric | Pooler (PgBouncer/RDS Proxy), smaller per-pod pools |
| **SYN backlog full** (server overwhelmed) | New connections time out; existing ones fine | `netstat -s` "SYNs to LISTEN sockets dropped"; `ListenOverflows` | Scale out, raise `somaxconn`/backlog, find slow accept loop |
| **Packet loss** | Latency spikes, throughput collapse | Retransmit counters (`nstat`, `ss -ti`), cloud network metrics | Fix bad link/NIC, avoid overloaded instances, spread across AZs |
| **MTU black hole** | Small requests work, big ones hang | `ping -M do -s 1472 host` fails | Lower MTU / clamp MSS on tunnels, allow ICMP type 3 code 4 |
| **Stale pooled connections** | Errors right after a server/LB restart or idle timeout | "connection reset by peer" bursts | Client idle timeout < server/LB idle timeout; retry idempotent requests once |
| **Cross-AZ / cross-region chattiness** | Slow requests, big data-transfer bill | Tracing shows many remote calls; cost explorer | Co-locate, batch calls, cache |
| **Conntrack table full** (Linux/K8s nodes) | Random packet drops: `nf_conntrack: table full, dropping packet` in `dmesg` | Node logs, `conntrack -C` | Raise `nf_conntrack_max`, reduce short-lived connections |

**3 AM:** "timeouts to the payment API spiked after traffic doubled". Check `ss -s` on a pod/node, NAT gateway port-allocation errors, and whether the HTTP client reuses connections. A one-line fix (enable keep-alive / shared client) often resolves it; the longer fix is a VPC endpoint or more NAT IPs.

---

## Where you'll use it

- **Placing components:** same AZ for chatty pairs where possible, same region always for synchronous paths.
- **Kubernetes:** Services, kube-proxy and conntrack ([K8s 32 · inside a Service](../../CKA-Certification-Course-2025/notes/32-inside-a-service-kube-proxy.md)); gRPC load balancing problems (topic 09).
- **Databases:** sizing connection pools; adding PgBouncer.
- **Docker:** port publishing and NAT on the host ([Docker 28 · publishing ports](../../Docker/notes/28-publishing-ports.md)).
- **Debugging:** `ss`, `tcpdump` ([Linux 096 · packet capture](../../linux-mastery-rhel/notes/096-packet-capture.md)), `mtr`, `curl -w`.

---

## Design exercise

**Prompt:** 400 pods each run an app with a DB connection pool of 25. The managed Postgres instance has `max_connections = 2000`. During a deploy (rolling, 25% surge), connection errors appear. Explain and fix.

<details>
<summary><b>Model answer</b></summary>

Steady state: 400 × 25 = **10,000** possible connections, already 5× the limit. It "worked" because pools open connections lazily and most pods used only a few. During a rolling deploy with 25% surge, up to 500 pods exist; new pods open connections while old ones still hold theirs, and warm-up traffic makes pools grow → the limit is hit → `too many connections`.

Fixes, in order:
1. **Size pools from the DB down:** budget e.g. 1,500 usable connections ÷ 500 max pods ≈ 3 per pod. Most apps need far fewer connections than threads, because queries take ~1–5 ms.
2. **Add a pooler** (PgBouncer in transaction mode, or RDS Proxy): thousands of client connections map onto ~200 server connections.
3. **Alert** at 70% of `max_connections`.
4. Long-term: if the DB genuinely needs more concurrency, scale it vertically or add read replicas for read traffic (topic 17).

Trade-off of PgBouncer transaction mode: session features (session-level prepared statements in older drivers, `SET` variables, advisory locks held across transactions) behave differently; test the app.
</details>

---

## Quick recap

- Each network call = round trips + failure risk + limits. Minimize calls and reuse connections.
- New HTTPS connection ≈ 3 RTTs before the first byte; latency ≠ bandwidth.
- TCP = reliable/ordered (head-of-line blocking); UDP = fast, no guarantees.
- Watch for port exhaustion (NAT, TIME_WAIT), DB connection limits, conntrack, MTU black holes.
- Pool connections on the client, and pool again in front of databases at scale.

### Interview questions

1. **Q: Why do connection pools matter so much in microservices on Kubernetes?**
   A: Each new TCP+TLS connection costs 2 RTTs and CPU for crypto, and creates TIME_WAIT sockets that consume ephemeral ports. With hundreds of pods, per-request connections cause latency, port exhaustion at NAT, and overwhelm databases with connection counts. Pools reuse warm connections; a server-side pooler protects the DB from the sum of all client pools.

2. **Q: A service is fast inside the region but slow for users in another continent, even though bandwidth is plenty. Why?**
   A: Latency, not bandwidth. Every new connection pays TCP + TLS handshakes at ~200 ms RTT, and slow start needs several RTTs to ramp up. Fixes: terminate connections close to users (CDN/edge, topic 15), keep connections warm, use HTTP/2 or 3, reduce round trips per page, or deploy in a nearer region.

3. **Q: When would you choose UDP?**
   A: When freshness beats completeness or the protocol adds its own reliability: DNS queries (small, retried by the client), voice/video (a late packet is useless), game state, high-volume telemetry where losing a sample is fine, and QUIC/HTTP/3 which implements reliability per stream on top of UDP to avoid TCP's head-of-line blocking.
