# 54 · Case Study: A Company-Wide Secrets Management Service

> **Level:** ★★★ advanced
>
> **You will practise:** designing **HashiCorp Vault (or OpenBao) as a shared platform service**: requirements, **HA with integrated Raft storage** across AZs, **auto-unseal with KMS**, **auth methods** (Kubernetes, AWS IAM, OIDC for humans, CI), **secrets engines** (KV, dynamic database creds, PKI, transit), **namespaces/tenancy and policies**, **performance and DR replication**, **client-side caching** so a Vault outage isn't a company outage, **audit**, upgrades, and "**Vault is down at 3 AM**".
>
> **You need:** topics 05, 17, 26, 37, 40–42.
>
> **Part 9 · Case studies** · Previous: [53 · Self-service infrastructure](53-case-self-service-infrastructure.md) · Next: [55 · Rate limiter / API gateway](55-case-rate-limiter-gateway.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design a central secrets management service for the whole company (cloud + on-prem)."

| Question | Assumed answer |
|---|---|
| Environments? | AWS (EKS, EC2), an on-prem data centre (VMs, some K8s), CI (GitHub Actions, Jenkins) |
| Consumers? | ~2,000 services/apps, ~1,500 engineers, CI pipelines, Ansible/Terraform |
| Secret types? | Static API keys, DB credentials (want dynamic), internal TLS certificates (PKI), encryption-as-a-service for apps |
| Load? | ~3,000 requests/s peak (logins, reads, renewals, cert issuance) |
| Availability? | 99.99% for reads/renewals in each site; secrets must be available even if the other site is down |
| Compliance? | Audit every access; separation of duties; keys in HSM-backed KMS |
| Why not cloud secret managers only? | Hybrid (on-prem + cloud), dynamic DB credentials across both, internal PKI, one policy model |

---

## 2. Estimation

```
Clients: 2,000 services × ~10 instances = 20,000 clients
Auth: tokens TTL 1 h, renewed → ~20,000 renewals/h ≈ 6/s ; logins at deploy waves: bursts of 1,000s/min
Dynamic DB creds: 5,000 instances × lease 1 h → ~1.4 creds/s steady, bursts on deploys
PKI: mesh/service certs (if Vault is the CA): 20,000 certs/day (24 h TTL) ≈ 0.25/s steady, bursts
KV reads with agent caching: ~1,000–3,000/s peak at Vault
Storage: secrets + leases + PKI metadata: tens of GB at most (lease count matters more than size)
```
**Conclusion:** throughput is moderate; the critical properties are **availability, low latency, lease management at scale, security of the root of trust, and graceful behaviour when Vault is unreachable**.

---

## 3. High-level design

```
                    PRIMARY CLUSTER (AWS ap-south-1, 5 nodes across 3 AZs)
Clients ─▶ NLB ─▶ Vault active node + standby nodes (performance standbys serve reads)
                    Integrated Storage (Raft), quorum 3 of 5
                    Auto-unseal: AWS KMS (HSM-backed key)
                    Audit devices: file → log shipper → SIEM (+ a second audit device; Vault blocks requests if ALL audit devices fail)
                    │
                    ├── Performance replication ──▶ ON-PREM CLUSTER (secondary: serves local reads, own leases & tokens)
                    └── DR replication ──────────▶ DR CLUSTER (AWS ap-south-2, warm, promotable)

Auth methods:   kubernetes (per cluster) │ aws (IAM) │ oidc (SSO for humans) │ jwt (GitHub Actions OIDC) │ approle (legacy)
Secrets engines: kv-v2 (static) │ database (dynamic creds: Postgres/MySQL) │ pki (intermediate CA for internal TLS) │ transit (encryption as a service)
Tenancy: namespaces per business unit or per team ; policies by path templates (identity-based)
Clients: Vault Agent / CSI provider / ESO for K8s ; SDKs ; Agent caching + templating ; renew leases
Config as code: Terraform vault provider manages mounts, auth roles, policies (reviewed PRs)
```

**Flow: a pod gets dynamic DB credentials**
1. Pod starts with ServiceAccount `orders` in cluster `prod-aps1`.
2. Vault Agent (sidecar/init) logs in via `auth/kubernetes-prod-aps1` with the SA token → Vault validates the token with that cluster's API (TokenReview) → returns a Vault token with policy `orders-prod`.
3. Agent reads `database/creds/orders-rw` → Vault creates a Postgres role with 1 h TTL → renders credentials to a file.
4. App reads the file; Agent renews the lease while the pod lives; on pod death, the lease expires and Vault drops the DB user.

---

## 4. Deep dives

### 4.1 HA and quorum
- 5 Raft voters over 3 AZs (2/2/1) → tolerates 2 node failures or 1 AZ (topic 26).
- One **active** node handles writes; **performance standbys** handle reads (Enterprise) or forward to active (OSS: standbys forward requests).
- NLB health check on `/v1/sys/health` (active vs standby status codes).
- Autopilot: dead server cleanup, stable server promotion, upgrade migrations.

### 4.2 Unseal and root of trust
- Vault data is encrypted by a **root key** that must be unsealed on start. **Auto-unseal with KMS**: the root key is wrapped by a KMS key; nodes call KMS at startup.
- Risk: KMS key deleted/disabled → Vault can't unseal → **total outage**. Protect with key policy, SCP denying `ScheduleKeyDeletion`/`DisableKey` except break-glass, and recovery keys (Shamir shares) held by separate officers.
- Root tokens: generated only for emergencies via recovery key quorum, revoked after use.

### 4.3 Policies and tenancy
```hcl
# policy template: a service can read only its own path, identified by K8s SA metadata
path "kv/data/prod/{{identity.entity.aliases.<k8s_accessor>.metadata.service_account_name}}/*" {
  capabilities = ["read"]
}
path "database/creds/{{identity.entity.aliases.<k8s_accessor>.metadata.service_account_name}}-rw" {
  capabilities = ["read"]
}
```
- Namespaces for business units (isolated mounts/policies, delegated admins).
- Humans: SSO groups → policies; write access to prod secrets only via approved workflow; no standing root.

### 4.4 Surviving Vault unavailability (the most important design point)
| Technique | Effect |
|---|---|
| Vault Agent **caching** + rendered files | Running apps keep working with current credentials during a Vault outage |
| Lease TTLs long enough (e.g. 1 h, max TTL 24 h) | Outage < TTL doesn't break running apps |
| K8s Secrets synced by ESO (for static secrets) | Pods can restart without Vault |
| Performance secondaries per site | On-prem doesn't depend on AWS link |
| Retry with backoff + jitter on login | No thundering herd when Vault returns |
| Avoid Vault on the per-request hot path | Transit encryption calls batched/cached data keys |

So: Vault outage = **can't start new pods that need new dynamic creds / can't rotate**, but running services continue.

### 4.5 Lease explosion
Each dynamic secret/token is a lease stored in Raft. Short TTLs × many clients × restarts can create millions of leases → slow Vault, big snapshots. Mitigate: reasonable TTLs, token reuse via Agent (don't log in per request), batch tokens for high-volume short-lived auth, monitor `vault.expire.num_leases`.

---

## 5. Scaling
- Vertical first (CPU for crypto, fast disks for Raft); 5 voters + non-voters for read scaling.
- Performance replication to add sites; client-side caching to cut read volume.
- PKI at high volume: short-lived certs with `no_store=true` (don't store every leaf cert), or delegate workload certs to the mesh CA (Vault as root/intermediate only).

---

## 6. Reliability and DR
| Failure | Effect | Response |
|---|---|---|
| One node | None (quorum holds) | Autopilot replaces |
| One AZ | 3 of 5 voters remain → OK | Watch |
| Active node bad / leader flapping | Request errors | Investigate disk latency (Raft fsync), CPU |
| Lost quorum | Vault unavailable (CP system) | Restore quorum or restore from Raft snapshot; apps continue on cached creds |
| Region loss | Primary gone | Promote DR secondary (DR operation token), repoint DNS; performance secondary on-prem keeps serving its site |
| KMS unavailable | Can't unseal restarted nodes (running ones keep working) | Don't restart nodes during KMS issues; multi-region KMS key for DR cluster |
| Bad policy change | Apps denied | Policies via Terraform PRs, tested; quick revert |

Backups: automated Raft snapshots every hour to S3 (cross-account, encrypted), restore tested monthly in an isolated environment.

---

## 7. Security
- TLS everywhere (Vault listeners with internal CA certs), mTLS optional for clients.
- Audit devices required (two), shipped to SIEM; alerts on root token use, policy changes, unusual read volumes, auth failures.
- Hardened nodes: dedicated instances, no SSH (SSM only), disk encryption, minimal access; **swap disabled** on the host so secrets in memory are never written to disk (with integrated Raft storage HashiCorp recommends `disable_mlock = true`, which makes disabling swap essential).
- Separation of duties: Vault operators ≠ secret owners; recovery key shares split among officers.

---

## 8. Observability
- Telemetry to Prometheus: request latency/errors by path, `vault.core.handle_request`, Raft commit time and leader changes, lease counts, token counts, audit log failures, seal status, replication lag (performance/DR WAL indexes).
- Synthetic: login + read a canary secret every minute from each site.
- SLOs: read/login availability 99.99% per site, p99 < 100 ms.

---

## 9. Cost
- Compute: 5 + 5 (DR) + on-prem nodes; modest.
- Licensing: Vault Enterprise needed for performance replication, namespaces, DR replication → major cost line; OSS/OpenBao alternatives lack some features (use cloud secret managers per environment + Vault OSS for PKI/dynamic secrets as a hybrid).
- People: a platform team owning it 24/7 (it's tier 0).

---

## 10. Trade-offs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Central Vault | One model for hybrid, dynamic secrets, PKI | Cloud secret managers per env | Hybrid + dynamic creds requirement; but higher ops burden |
| Integrated Raft | Fewer moving parts | Consul storage | Simpler ops |
| Agent caching + long-ish TTLs | Resilience to Vault outages | Very short TTLs | Short TTL reduces leak window but increases Vault dependency and leases |
| Auto-unseal KMS | Automatic restarts | Shamir manual unseal | Operational availability; KMS becomes critical dependency (protected) |

---

## 11. Follow-up questions

1. **Q: Vault is down at 3 AM. What breaks and what do you do?**
   A: Running services keep working (cached/rendered creds valid until lease expiry); new pods needing new dynamic creds fail to start; rotations pause. Check quorum/leader, seal status, KMS, disk latency. Restore quorum or fail over to DR. If the outage will outlast lease TTLs, extend TTLs on the DR cluster or temporarily fall back to static credentials per the runbook.

2. **Q: Why dynamic DB credentials instead of a shared password in Vault KV?**
   A: Each instance gets unique, short-lived credentials: leaks are contained and expire automatically, access is attributable per instance, and revocation is instant. Static KV passwords need manual rotation and are shared.

3. **Q: How do CI pipelines authenticate to Vault without a stored secret?**
   A: GitHub Actions OIDC tokens validated by Vault's JWT auth method with bound claims (repo, branch, environment), giving a short-lived Vault token with a policy scoped to that pipeline. No secret-zero stored in CI.

4. **Q: How do you upgrade Vault safely?**
   A: Read release notes, upgrade DR/standby clusters and non-prod first, take a Raft snapshot, upgrade standbys one at a time then step down the active (Autopilot upgrade migrations help), verify health and replication, keep rollback snapshots, and avoid upgrading during KMS or network instability.
