# 07 · DNS for System Design

> **Level:** ★★ intermediate
>
> **You will learn:** how a name becomes an IP (**recursive resolver, root, TLD, authoritative**), the record types you design with (**A, AAAA, CNAME, ALIAS, NS, MX, TXT, SRV**), **TTL** and its trade-off between speed of change and load/resilience, **DNS-based traffic routing** (weighted, latency, geo, failover with health checks), **anycast**, **private DNS** inside VPCs and Kubernetes, and why DNS is a classic **single point of failure**.
>
> **You need:** topics 04–06. [Linux 063 · name resolution](../../linux-mastery-rhel/notes/063-name-resolution-and-dns-client.md), [Linux 081 · BIND](../../linux-mastery-rhel/notes/081-dns-server-bind.md) and [K8s 33 · CoreDNS](../../CKA-Certification-Course-2025/notes/33-cluster-dns-and-coredns.md) go hands-on.
>
> **Part 1 · Networking and the edge** · Previous: [06 · HTTP](06-http-and-api-styles.md) · Next: [08 · Cloud network building blocks](08-cloud-network-building-blocks.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **DNS** (Domain Name System) is the internet's phone book: `shop.example.com → 203.0.113.10`. It's also a **traffic-steering tool**: by answering differently per user, region or health, DNS sends users to the right place.

```
Browser → OS cache → recursive resolver (ISP / 8.8.8.8 / VPC resolver)
            → root (.) → TLD (.com) → authoritative (example.com's DNS provider) → answer + TTL
Answer is cached for TTL seconds at every layer.
```

**Picture it.** Directory enquiries. You ask the operator (resolver). They don't know, so they ask the country directory (root), then the city directory (TLD), then the company's receptionist (authoritative), and write the number on a sticky note valid for a while (TTL).

**The one thing to know.** **TTL** is a trade-off. Low TTL (30–60 s) = fast failover and changes, more queries. High TTL (hours) = fewer queries and resilience if DNS is briefly down, but changes take hours to reach everyone. And some clients ignore TTL and cache longer, so **DNS failover is never instant**.

**Try it now:**

```bash
dig +noall +answer www.amazon.com           # follow the CNAME chain, see TTLs
dig +trace example.com                        # root → TLD → authoritative, step by step
```

**You've got it if you can:** walk through a resolution, pick a TTL with a reason, and explain DNS failover's limits.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Part | Job |
|---|---|
| **Stub resolver** | The tiny client in your OS (`/etc/resolv.conf` on Linux) that asks a recursive resolver. |
| **Recursive resolver** | Does the work of finding the answer and caches it. ISP's, public (8.8.8.8, 1.1.1.1), or your cloud's (AWS VPC resolver at `VPC CIDR base + 2`, e.g. `10.0.0.2`). |
| **Root servers** | Know where each TLD's servers are. 13 names, hundreds of anycast instances. |
| **TLD servers** | `.com`, `.in`, `.io`. Know which servers are authoritative for `example.com`. |
| **Authoritative servers** | Hold the real records for your zone. AWS Route 53, Cloudflare, Google Cloud DNS, Azure DNS, NS1. |
| **Zone** | The set of records for a domain you manage (e.g. `example.com`). |

**Record types:**

| Record | Maps | Example / use |
|---|---|---|
| **A** | name → IPv4 | `shop.example.com A 203.0.113.10` |
| **AAAA** | name → IPv6 | |
| **CNAME** | name → another name (alias) | `www → shop-lb-123.ap-south-1.elb.amazonaws.com`. Not allowed at the zone apex (`example.com` itself). |
| **ALIAS / ANAME** (provider feature) | apex → a cloud resource name, resolved by the provider | Route 53 **Alias** to an ALB/CloudFront at `example.com`. Free queries in Route 53. |
| **NS** | which servers are authoritative | Delegation from the parent zone |
| **MX** | mail servers | |
| **TXT** | free text | Domain verification, SPF/DKIM (email auth), ACME DNS-01 challenge (topic 05) |
| **SRV** | service → host + port | Some service discovery (Kubernetes headless services, some databases) |
| **CAA** | which CAs may issue certs for you | Security hardening |

**TTL** (Time To Live) = seconds a resolver may cache the answer.

---

## Why it exists

Humans and configs use names; machines need IPs. Names also let you **move** things (change the IP behind a name without touching clients) and **steer** traffic (answer different IPs for different users). In design, DNS is:

1. The **first hop** of every request (and a single point of failure if done badly).
2. A **global load balancer** for multi-region (topic 42).
3. The **service discovery** layer inside Kubernetes and VPCs.

---

## Analogy

Directory enquiries (Core). Extensions:

- **TTL** = how long you trust the sticky note before calling again.
- **Weighted routing** = the receptionist sends 90% of callers to branch A, 10% to branch B.
- **Failover routing** = if branch A doesn't answer its test call, the receptionist gives out branch B's number.
- **Clients ignoring TTL** = people who keep the old sticky note on the fridge for months.

---

## How it works inside

### Resolution step by step

```
 1. App calls getaddrinfo("shop.example.com")
 2. OS checks its cache / /etc/hosts (and nscd/systemd-resolved if present)
 3. Stub asks recursive resolver (e.g. VPC resolver 10.0.0.2)
 4. Resolver cache hit? → answer immediately (~1 ms)
    Miss → ask root:      "who handles .com?"         → NS list for .com
 5.        ask .com TLD:  "who handles example.com?"   → NS = ns-1.awsdns-01.org ...
 6.        ask authoritative: "shop.example.com?"      → CNAME shop-lb-123.elb.amazonaws.com
 7.        resolve that name the same way              → A 13.x.x.x (TTL 60)
 8. Resolver caches each answer for its TTL, returns to the stub
```

Uncached, this takes 20–100+ ms; cached, ~1 ms. That's why resolvers and caches matter.

### DNS routing policies (Route 53 names; others are similar)

| Policy | How it answers | Use for |
|---|---|---|
| **Simple** | Same answer for everyone | Single endpoint |
| **Weighted** | Splits answers by weight (90/10) | Blue/green, canary across stacks, migrations |
| **Latency-based** | Region with lowest measured latency for the resolver | Multi-region active-active |
| **Geolocation** | By the user's country/continent | Data residency, localized content |
| **Geoproximity** | By distance, with bias | Shifting load between regions |
| **Failover** | Primary while its **health check** passes, else secondary | Active-passive DR (topic 41) |
| **Multivalue** | Several healthy IPs, client picks | Simple client-side spread |

**Health checks:** the DNS provider probes your endpoint from several locations (e.g. every 10–30 s) and stops returning unhealthy answers.

### Anycast

The same IP address is announced from many locations worldwide; the internet's routing (**BGP**, Border Gateway Protocol, how networks tell each other which IPs they can reach) sends each user to the nearest one. Used by root servers, public resolvers, CDNs, and DNS providers themselves. It gives resilience and low latency **without** depending on DNS TTLs.

### Private DNS

- **VPC private hosted zones** (Route 53 private zones; GCP Cloud DNS private zones; Azure Private DNS): names that resolve only inside your VPCs, e.g. `db.internal.example.com`.
- **Kubernetes:** CoreDNS answers `my-svc.my-ns.svc.cluster.local` with the Service's ClusterIP. Pods have `ndots:5` in `resolv.conf`, which makes every external lookup try several cluster suffixes first, multiplying DNS queries. ([K8s 33](../../CKA-Certification-Course-2025/notes/33-cluster-dns-and-coredns.md).)
- **Hybrid:** Route 53 Resolver inbound/outbound endpoints forward queries between on-prem DNS and VPCs.

---

## Options and trade-offs

### TTL choice

| TTL | Pros | Cons | Use for |
|---|---|---|---|
| 30–60 s | Fast failover/changes | More queries ($ per million), more resolver load, more exposure to DNS outages | Records you may fail over or shift (LB endpoints in multi-region) |
| 5–15 min | Balanced | Changes take minutes | Most records |
| Hours–1 day | Cheap, resilient to brief DNS outages | Slow changes, slow rollback | NS records, rarely changed records |

**Migration trick:** lower the TTL to 60 s **a day before** a planned change (wait for the old TTL to expire), make the change, then raise it again.

### DNS-based vs other global traffic steering

| Method | Failover speed | Granularity | Pros | Cons |
|---|---|---|---|---|
| DNS failover/latency routing | TTL + health-check interval + client caching: **1–5+ min** in practice | Per resolver (not per user) | Cheap, simple, works with anything | Clients ignore TTL; resolver location ≠ user location (EDNS Client Subnet helps) |
| Anycast global LB (AWS Global Accelerator, GCP global LB, Azure Front Door, Cloudflare) | Seconds | Per connection | Fast failover, static IPs | Cost, vendor features |
| CDN with multiple origins (origin failover) | Seconds | Per request | Built into CDN | Only for traffic through the CDN |

### DNS provider strategy

| Option | Pros | Cons |
|---|---|---|
| One managed provider (Route 53: 100% availability SLA) | Simple, integrated | Provider-wide outage = you're down |
| Two providers (primary + secondary, NS records from both) | Survives one provider's outage | Must keep zones in sync (OctoDNS, Terraform for both) and lose provider-specific routing features unless both support them |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Wrong record pushed** | Site unreachable or pointing at the wrong place | External synthetic checks, `dig` from many locations | DNS as code (Terraform) with review; low TTL before risky changes |
| **DNS provider outage** (it happens, e.g. the 2016 Dyn DDoS) | Names don't resolve for many users | Synthetic checks from outside | Secondary provider; higher TTLs on stable records |
| **Domain registration / NS delegation expired** | Whole domain gone | Registrar expiry alerts | Auto-renew, registry lock, monitor NS delegation |
| **Clients cache too long (JVM, some SDKs)** | Failover done but some clients still hit old IP for hours | Traffic still arriving at old endpoint | Configure client TTL (Java `networkaddress.cache.ttl`), use anycast/static IPs |
| **Resolver overload / throttling** | Intermittent `SERVFAIL`/timeouts; in AWS each ENI is limited (~1,024 packets/s to the VPC resolver) | Resolver error metrics; app DNS timeouts | Node-local DNS cache (K8s NodeLocal DNSCache), reduce `ndots`, connection reuse |
| **CoreDNS overloaded** | Random service-to-service failures in K8s | CoreDNS latency/error metrics | Scale CoreDNS, NodeLocal DNSCache, FQDN with trailing dot |
| **Negative caching** | Created a record but it still "doesn't exist" | `dig` shows NXDOMAIN from resolver | Wait SOA negative TTL; don't query names before creating them |
| **Split-horizon confusion** | Works inside VPC, fails outside (or vice versa) | Compare `dig` inside vs outside | Document private vs public zones |
| **Dangling CNAME** (points to a deleted cloud resource) | Subdomain takeover by an attacker | Security scans | Remove records with resources (manage both in the same Terraform) |

**3 AM:** "Half of users can't reach us after the failover." DNS failover is working but clients/resolvers still cache the old answer (TTL, or ignored TTL). Immediate: keep the old endpoint serving if possible (or redirect it); longer term: anycast/global accelerator for sub-minute failover, lower TTLs, client DNS settings.

---

## Where you'll use it

- **Terraform:** `aws_route53_record` with alias to ALB, health checks, failover/weighted policies ([Terraform 32](../../Terraform/notes/32-aws-networking.md)).
- **Kubernetes:** service discovery via CoreDNS; ExternalDNS controller creating Route 53 records from Ingress/Service objects.
- **Blue/green and migrations:** weighted records shifting traffic gradually (topic 34).
- **DR:** failover records for active-passive (topic 41).
- **Cert automation:** DNS-01 challenges via TXT records (topic 05).

---

## Design exercise

**Prompt:** You run `api.example.com` in `ap-south-1` (primary) and `ap-southeast-1` (standby, warm). Requirement: users should reach the standby within 5 minutes of a primary region failure. Design the DNS part and explain how long failover really takes.

<details>
<summary><b>Model answer</b></summary>

- Route 53 **failover routing**: primary record → ALB in `ap-south-1` (alias), secondary → ALB in `ap-southeast-1`.
- **Health check** on the primary: HTTPS `GET /healthz` (a deep-ish check that verifies the app can reach its DB, but not every dependency, to avoid false failovers), interval 10 s, failure threshold 3 → detection ≈ 30 s. Or a CloudWatch-alarm-based health check on error rate.
- **TTL:** alias records to ALBs use the ALB's TTL (60 s).
- **Time to failover:** detection ~30 s + DNS propagation (resolvers' cached answers expire within the 60 s TTL) + clients that cache longer. Most users: **~1.5–2 minutes**. Some clients ignoring TTL: longer.
- To meet 5 minutes **reliably**: also configure clients (mobile SDKs, Java TTL), or put **AWS Global Accelerator** in front (anycast static IPs, regional failover in ~30 s, not dependent on DNS caching).
- DNS is only the front door: the standby must actually be able to serve (data replicated, capacity scaled, topic 41–42). Test with a game day (topic 44).
- Failback: manual, after the primary is verified healthy and data is re-synced, to avoid flapping.
</details>

---

## Quick recap

- Resolution: stub → recursive resolver → root → TLD → authoritative; cached per TTL at every step.
- Records: A/AAAA, CNAME (not at apex), ALIAS (provider apex alias), NS, MX, TXT, SRV, CAA.
- TTL trades change speed against load and resilience; lower it before planned changes.
- DNS routing: weighted, latency, geo, failover with health checks. DNS failover takes minutes and some clients ignore TTL.
- Anycast/global LBs give faster failover; DNS (provider, registration, delegation) is a SPOF to plan for.

### Interview questions

1. **Q: Why isn't DNS failover instant, and how do you make failover faster?**
   A: Detection takes health-check interval × threshold; then resolvers serve cached answers until TTL expires, and some clients cache longer than TTL. Faster options: anycast-based global load balancers or accelerators with static IPs, CDN origin failover, short TTLs plus correctly configured clients.

2. **Q: Why can't you put a CNAME at `example.com`?**
   A: The zone apex must hold SOA and NS records, and a CNAME can't coexist with other records at the same name. Providers offer ALIAS/ANAME records that resolve the target internally and return A records, e.g. Route 53 Alias to an ALB.

3. **Q: Pods in your cluster see intermittent DNS timeouts under load. What's going on?**
   A: Likely resolver limits or CoreDNS saturation, amplified by `ndots:5` making each external lookup try several search domains, plus conntrack races with UDP. Fixes: NodeLocal DNSCache, scale CoreDNS, lower `ndots` or use FQDNs with a trailing dot, reuse connections so fewer lookups happen, and watch AWS's per-ENI resolver packet limit.
