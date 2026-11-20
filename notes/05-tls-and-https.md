# 05 · TLS and HTTPS

> **Level:** ★ foundation
>
> **You will learn:** what **TLS** gives you (encryption, integrity, identity), how the **handshake** works and what it costs, **certificates and chains**, **where to terminate TLS** (edge, load balancer, pod, end to end), **mTLS** for service-to-service identity, **certificate automation** (ACME, cert-manager, ACM), and the most common TLS outage of all: **an expired certificate**.
>
> **You need:** topic 04. Your [Linux 064 · cryptography basics](../../linux-mastery-rhel/notes/064-cryptography-basics.md) and [K8s 51 · certificates and CAs](../../CKA-Certification-Course-2025/notes/51-certificates-and-cas.md) cover the crypto building blocks.
>
> **Part 1 · Networking and the edge** · Previous: [04 · Networking](04-networking-essentials.md) · Next: [06 · HTTP and API styles](06-http-and-api-styles.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **TLS** (Transport Layer Security) wraps a TCP connection so nobody can **read** it (encryption), **change** it (integrity) or **pretend** to be the server (identity via certificates). **HTTPS** = HTTP over TLS.

```
Client ── TCP handshake (1 RTT) ──▶ Server
Client ── TLS 1.3 handshake (1 RTT): "hello, my key share" / "here's my cert + key share" ──▶
Both sides now share a secret key → encrypted HTTP flows
```

**Picture it.** A sealed, tamper-proof envelope (encryption + integrity) with a passport photo checked at the door (certificate signed by a trusted authority).

**The one thing to know.** **Where you terminate TLS** (decrypt it) is a design decision: at the CDN/load balancer (simple, lets the LB see HTTP) or all the way to the pod (end-to-end encryption, needed for strict compliance). And **automate certificate renewal**; expired certs are among the most common causes of total outages.

**Try it now:**

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

You'll see who the cert is for, who signed it, and when it expires.

**You've got it if you can:** list the three things TLS gives you, explain termination options and their trade-offs, and design certificate renewal so it can't silently expire.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

- **TLS** = the protocol that secures connections. Successor of SSL (Secure Sockets Layer, now insecure; people still say "SSL certificate"). Use **TLS 1.2 or 1.3**; disable 1.0/1.1.
- **Certificate** (X.509 cert) = a file saying "this public key belongs to `shop.example.com`", **signed** by a **CA** (Certificate Authority) the client trusts.
- **Chain of trust**: your cert is signed by an **intermediate CA**, which is signed by a **root CA** pre-installed in browsers/OS. The server must send its cert **plus the intermediate(s)**.
- **SNI** (Server Name Indication) = the client says which hostname it wants during the handshake, so one IP can serve many sites with different certs.
- **mTLS** (mutual TLS) = **both** sides present certificates. The server verifies the client too. Used for service-to-service identity.
- **TLS termination** = the point where encrypted traffic is decrypted.

---

## Why it exists

Without TLS, anyone on the path (café Wi-Fi, an ISP, a compromised router, another tenant on a shared network) can read passwords and change responses. Regulations (PCI-DSS for cards, HIPAA for health data in the US, GDPR) require encryption in transit. Browsers mark HTTP sites "Not secure". HTTP/2 in browsers requires TLS.

Inside data centres, "the network is trusted" is no longer assumed (zero trust, topic 40): service-to-service traffic increasingly uses mTLS.

---

## Analogy

**A diplomatic pouch.**

- Sealed so nobody can read it in transit (**encryption**).
- Tamper-evident seal: if opened, you know (**integrity**).
- The courier shows a passport issued by a government you trust (**certificate signed by a CA**).
- With **mTLS**, the embassy receiving it also checks the *sender's* credentials.
- **TLS termination at a load balancer** = the pouch is opened at the embassy gate and the letter is walked inside by staff. Fine if the inside is trusted; not fine if regulations say "only the ambassador may open it" (end-to-end encryption).

---

## How it works inside

### TLS 1.3 handshake (simplified)

```
Client                                              Server
  │ ClientHello: TLS versions, cipher suites,          │
  │   SNI=shop.example.com, key share (ECDHE)  ───────▶│
  │                                                    │ picks cipher, makes its key share
  │◀────── ServerHello + key share, {Certificate,      │
  │        CertificateVerify (signature), Finished}    │  (already encrypted)
  │ verify cert chain + hostname + signature           │
  │ {Finished} + first HTTP request  ─────────────────▶│
  │◀───────────────────────────────── HTTP response    │
```

1. Client sends supported versions/ciphers, the hostname (SNI), and its half of a **key exchange** (ECDHE: Elliptic-Curve Diffie-Hellman Ephemeral, which lets two sides agree on a secret over an open channel).
2. Server replies with its half, its certificate chain, and a signature proving it owns the cert's private key.
3. Client verifies: chain leads to a trusted root, cert not expired, hostname matches, signature valid.
4. Both derive the same **session keys**; everything after is encrypted with fast symmetric crypto (AES-GCM or ChaCha20).

**Cost:**
- TLS 1.3: **1 RTT** (TLS 1.2 needed 2). **0-RTT resumption** exists for repeat visitors but replayable, so only for safe idempotent requests.
- CPU: the asymmetric part (handshake) is the expensive bit; bulk encryption is cheap on modern CPUs with AES instructions. → **reuse connections** (topic 04).
- **Forward secrecy**: ephemeral keys mean a stolen server key later can't decrypt recorded old traffic. TLS 1.3 always has it.

### Where to terminate TLS

```
Option A: terminate at edge/LB             Option B: re-encrypt              Option C: passthrough (end-to-end)
User ═TLS═▶ LB ──HTTP──▶ pod                User ═TLS═▶ LB ═TLS═▶ pod          User ═TLS═════════════▶ pod
           (decrypts, sees HTTP)                       (decrypts, inspects,              (LB at L4, can't see HTTP)
                                                        re-encrypts)
```

### Certificate automation

- **ACME** (Automatic Certificate Management Environment) = the protocol Let's Encrypt uses: prove you control a domain (HTTP-01 file or DNS-01 TXT record), get a cert for 90 days, renew automatically.
- **Cloud-managed:** AWS **ACM** (Certificate Manager) issues and auto-renews certs for ALB/CloudFront for free. GCP: Google-managed certificates / Certificate Manager. Azure: App Service managed certs / Key Vault.
- **Kubernetes:** **cert-manager** runs ACME or an internal CA and stores certs as Secrets for Ingress/Gateway ([K8s 71 · Ingress TLS](../../CKA-Certification-Course-2025/notes/71-ingress-tls-and-host-routing.md)).
- **Internal PKI** (Public Key Infrastructure: your own CA) for mTLS: HashiCorp Vault PKI, AWS Private CA, a service mesh's built-in CA (Istio/Linkerd rotate workload certs every ~24 h).
- ⚠️ Public CA certificate lifetimes are being shortened by the CA/Browser Forum in stages (toward ~47 days by 2029). Manual renewal will become impossible; automation is mandatory.

---

## Options and trade-offs

| Termination point | Pros | Cons | Choose when |
|---|---|---|---|
| **At CDN/LB, plain HTTP inside** | Simplest; LB can route by path/header, add WAF; one place for certs | Traffic inside VPC unencrypted | Low-sensitivity internal traffic, early stage (still common) |
| **Re-encrypt LB → backend** | Encrypted everywhere, LB still sees HTTP | Two cert sets; backend certs to manage | Compliance needs "encrypted in transit everywhere" (most regulated setups) |
| **Passthrough to pod (L4)** | True end-to-end; LB never sees plaintext | No L7 routing/WAF at LB; certs on every backend | Strict requirements, or protocols the LB doesn't understand |
| **Service mesh mTLS** | Automatic identity + encryption between all pods, cert rotation handled | Mesh complexity, sidecar/ambient overhead (topic 10) | Many services, zero-trust requirements |

| Cert source | Pros | Cons |
|---|---|---|
| Cloud-managed (ACM etc.) | Free, auto-renewed, private key never leaves cloud | Only works on that cloud's LB/CDN |
| Let's Encrypt via cert-manager | Free, automated, portable | Rate limits; ACME challenges must work (DNS/HTTP access) |
| Commercial CA (OV/EV) | Org validation, support | Cost, often manual renewal ⚠️ |
| Internal CA | Full control for internal/mTLS | You run a CA: protect root key, distribute trust bundles |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Expired certificate** | Every client fails: `certificate has expired`; full outage | Synthetic check of days-to-expiry; `probe_ssl_earliest_cert_expiry` (Prometheus blackbox exporter) alert at 21 and 7 days | Automate renewal; alert on expiry for *every* endpoint incl. internal and client certs |
| **Missing intermediate cert** | Works in Chrome (it fetches/caches intermediates), fails in curl/Java/mobile | `openssl s_client -showcerts`; SSL Labs test | Serve the full chain (`fullchain.pem`) |
| **Hostname mismatch** | `certificate is not valid for name` | Same as above | Correct SAN (Subject Alternative Name) list; wildcard if appropriate |
| **Renewal works but app didn't reload** | New cert on disk, old one served → expires anyway | Expiry probe on the **live endpoint**, not the file | Reload on change (cert-manager + reloader, nginx reload hook) |
| **ACME challenge blocked** | Renewal fails silently for weeks | cert-manager `Certificate` not Ready; events | Alert on renewal failure, not only on expiry |
| **Expired internal CA / mTLS root** | All service-to-service calls fail at once | Expiry monitoring of CA certs | Long-lived roots with planned rotation, overlapping trust bundles |
| **Clock skew** | "certificate not yet valid" on some hosts | NTP monitoring | chrony/NTP everywhere ([Linux 069](../../linux-mastery-rhel/notes/069-time-sync-chrony.md)) |
| **TLS CPU overload** | High CPU on LB/proxy during traffic spikes or reconnect storms | CPU, handshake rate metrics | Keep-alive, session resumption, offload to managed LB/CDN |

**3 AM:** "Everything is down, 100% of requests fail, no deploy happened." Check certificate expiry first. It's one command (`openssl s_client ... | openssl x509 -noout -dates`) and it's a very common cause of total outages with no change. Immediate fix: issue/upload a new cert (ACM, cert-manager force renew); postmortem: add expiry alerts on every endpoint.

---

## Where you'll use it

- **Ingress/Gateway** TLS on Kubernetes with cert-manager ([K8s 71](../../CKA-Certification-Course-2025/notes/71-ingress-tls-and-host-routing.md)).
- **Load balancer listeners** in Terraform (`aws_lb_listener` with `certificate_arn` from `aws_acm_certificate`).
- **Kubernetes internals** use mTLS everywhere: API server ↔ kubelet ↔ etcd ([K8s 53](../../CKA-Certification-Course-2025/notes/53-tls-inside-the-cluster.md)).
- **Reverse proxies** with nginx ([Docker 39a · TLS and reverse proxies](../../Docker/notes/39a-tls-and-reverse-proxies.md)).
- **Compliance answers:** "Is data encrypted in transit?" needs a clear termination diagram.

---

## Design exercise

**Prompt:** A fintech company runs 40 microservices on EKS behind an ALB. The auditor requires "all traffic encrypted in transit, including inside the VPC" and "the ability to prove which service called which". Design the TLS setup.

<details>
<summary><b>Model answer</b></summary>

1. **Edge:** ACM certificate on the ALB (auto-renewed). TLS 1.2+ security policy. AWS WAF on the ALB.
2. **ALB → cluster:** re-encrypt. The ALB sends HTTPS to the ingress gateway, which has a cert from cert-manager (internal CA).
3. **Service to service:** a **service mesh with automatic mTLS** (Istio or Linkerd) in strict mode. Every workload gets a short-lived certificate tied to its Kubernetes ServiceAccount identity (SPIFFE ID format, e.g. `spiffe://cluster.local/ns/payments/sa/api`). This satisfies "encrypted everywhere" and "prove who called whom": the mesh logs the caller identity, and **authorization policies** can allow only `orders → payments`.
4. **Databases:** RDS with `rds.force_ssl=1`; apps verify the RDS CA bundle.
5. **Monitoring:** expiry alerts on ALB cert (ACM also emits expiry events), ingress certs (cert-manager metrics), mesh root CA. Alert on cert-manager renewal failures.

Trade-offs: mesh adds operational complexity and some latency (~1–3 ms per hop with sidecars, less in ambient/eBPF modes) and CPU. The alternative (each app doing its own mTLS with Vault-issued certs) is cheaper at runtime but much harder to do consistently across 40 services in different languages.
</details>

---

## Quick recap

- TLS = encryption + integrity + server identity (and client identity with mTLS).
- TLS 1.3 adds 1 RTT; reuse connections to avoid repeated handshakes.
- Termination: edge only (simple), re-encrypt (compliant), passthrough (end-to-end, no L7 at LB), mesh mTLS (identity + encryption between services).
- Automate certificates (ACM, cert-manager, ACME) and alert on expiry of the *live* endpoint.
- Always serve the full chain; watch clock skew.

### Interview questions

1. **Q: Where would you terminate TLS for a public web app on Kubernetes, and why?**
   A: At the cloud load balancer or CDN with a managed cert, because it's simplest and enables L7 routing and WAF. If compliance requires encryption inside the VPC, re-encrypt from LB to the ingress, and use mesh mTLS between services. Passthrough only when the LB must never see plaintext; then I lose L7 features at the LB.

2. **Q: How do you make sure certificate expiry never causes an outage?**
   A: Automate issuance and renewal (ACM, cert-manager with ACME or an internal CA), make apps reload certs automatically, and monitor expiry on the **live endpoints** (blackbox probes) with alerts weeks ahead, plus alerts on renewal failures. Include internal CAs and client certs in the inventory; those are the ones people forget.

3. **Q: What does mTLS give you that network policies don't?**
   A: Network policies decide which IPs/pods can connect (layer 3/4). mTLS gives cryptographic **identity** for each workload, encryption, and lets you authorize by identity rather than IP, which survives pod rescheduling and works across clusters. They complement each other: defence in depth.
