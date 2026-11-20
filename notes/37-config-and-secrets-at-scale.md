# 37 · Configuration and Secrets at Scale

> **Level:** ★★★ advanced
>
> **You will learn:** the difference between **config** and **secrets**, where config should live (env vars, files, ConfigMaps, a config service) and how config changes are **deployed safely**, **secret managers** (Vault, AWS Secrets Manager/SSM, GCP Secret Manager, Azure Key Vault), **workload identity** as the secret-zero answer, **dynamic secrets** and **rotation**, **envelope encryption** with **KMS**, delivering secrets to **Kubernetes** (External Secrets Operator, CSI driver, Vault agent, SOPS/Sealed Secrets for GitOps), and the failure modes (leaks, rotation outages, secret manager as SPOF).
>
> **You need:** topics 05, 32–36. [K8s 35 · Secrets](../../CKA-Certification-Course-2025/notes/35-secrets.md), [Ansible 41 · external secrets](../../Ansible/notes/41-external-secrets.md) and [Terraform 51 · secrets](../../Terraform/notes/51-secrets.md) are the tool-level views.
>
> **Part 7 · Infrastructure and platform** · Previous: [36 · IaC at scale](36-iac-at-scale.md) · Next: [38 · Kubernetes platform design](38-kubernetes-platform-design.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.**
```
Config  = settings that change behaviour, not dangerous if seen    (log level, URLs, timeouts, feature flags)
Secret  = anything that grants access if leaked                    (DB passwords, API keys, TLS private keys, tokens)
```
Secrets live in a **secret manager**, encrypted with keys in a **KMS** (Key Management Service). Workloads authenticate with their **identity** (IAM role for the pod / VM), not with another secret. Best of all: **short-lived, dynamic** credentials that expire on their own.

```
Pod (ServiceAccount orders) ──workload identity──▶ AWS IAM role orders-prod
        └──▶ Secrets Manager GetSecretValue("prod/orders/db") or Vault: "give me a DB login" (valid 1 h)
```

**Picture it.** A hotel. Config = the welcome booklet (anyone can read it). Secrets = room keys. A good hotel issues **key cards** that expire at checkout (dynamic secrets), checks your ID before issuing them (workload identity), and keeps the master key in a safe (KMS). A bad hotel leaves metal keys under the doormat (secrets in Git).

**The one thing to know.** The hardest problem is **"secret zero"**: how does an app prove who it is to get its first secret? The answer in the cloud is **workload identity** (IRSA/EKS Pod Identity, GKE Workload Identity, Azure Workload Identity, instance profiles), so there's no bootstrap secret to leak.

**Try it now:** `git log -p | grep -iE "password|secret|api_key|BEGIN RSA" | head` on a repo. Then run a real scanner (gitleaks, trufflehog).

**You've got it if you can:** design secret delivery for K8s with GitOps, explain dynamic secrets and rotation, and explain envelope encryption.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Concept | Meaning |
|---|---|
| **Secret manager** | Service storing secrets encrypted, with access control, audit and rotation |
| **KMS** | Managed service holding encryption keys in hardware security modules (HSMs); keys never leave it |
| **Envelope encryption** | Data encrypted with a **data key**; the data key is encrypted with a **KMS master key**; store encrypted data + encrypted data key together |
| **Workload identity** | Cloud identity bound to a workload (pod ServiceAccount → IAM role) via short-lived tokens |
| **Static secret** | Long-lived value (an API key) stored and rotated periodically |
| **Dynamic secret** | Created on demand per client with a TTL (Vault DB engine creates a DB user valid 1 h) |
| **Rotation** | Replacing a secret with a new one on a schedule or after exposure |
| **Secret sprawl** | Copies of secrets in many places (CI vars, env files, wikis, chat) |

| | AWS | GCP | Azure | Neutral |
|---|---|---|---|---|
| Secrets | Secrets Manager, SSM Parameter Store (SecureString) | Secret Manager | Key Vault | HashiCorp Vault / OpenBao, Infisical |
| Keys | KMS, CloudHSM | Cloud KMS, Cloud HSM | Key Vault (keys), Managed HSM | Vault Transit |
| Workload identity | IRSA, EKS Pod Identity, instance profiles | Workload Identity Federation | Managed identities, Workload Identity | SPIFFE/SPIRE |

---

## Why it exists

- Leaked credentials are a leading cause of breaches (keys in Git, CI logs, Docker images, Terraform state).
- Long-lived shared passwords can't be traced to who used them and are painful to rotate.
- Config sprawl across hundreds of services causes inconsistency and risky manual changes.
- Compliance requires encryption at rest, access audit and rotation.

---

## Analogy

Hotel key cards (Core). **Envelope encryption** = documents locked in a box with a cheap padlock (data key); the padlock's key is kept in the bank vault (KMS). To open, you ask the bank to unlock the small key, never taking the vault's master key out. Rotating the master key doesn't require re-locking every box (re-wrap data keys).

---

## How it works inside

### Configuration management

| Where | Good for | Notes |
|---|---|---|
| Env vars (12-factor) | Simple per-deploy settings | Change = restart; visible in process listings/`describe` |
| Files (ConfigMap mounted) | Larger config, reload without restart (if app watches file) | ConfigMap updates propagate to mounted files in ~1 min; env vars from ConfigMaps don't update |
| Config in Git (GitOps) | Auditable, reviewed, promoted per env | Topic 33 |
| Dynamic config service (AppConfig, Consul KV, flag systems) | Runtime changes without deploy (flags, limits) | Must be rolled out progressively, validated, cached (topic 57) |

⚠️ **Config changes cause outages** as often as code changes (a wrong timeout, a typo in a routing rule pushed globally). Treat them like code: review, validate (schema), progressive rollout, easy rollback. Hash the ConfigMap into the pod template annotation so a change triggers a rolling restart (Helm/Kustomize patterns).

### Secret delivery patterns

```
1. App calls secret manager SDK at startup/runtime (using workload identity)
   + no K8s Secret object; caching and refresh in app  − code changes, SDK per language

2. External Secrets Operator (ESO): syncs secret manager → K8s Secret
   ExternalSecret CR (in Git, no values) → ESO fetches → creates K8s Secret → pod env/volume
   + GitOps-friendly, apps unchanged  − values exist in etcd (encrypt etcd with KMS) 

3. Secrets Store CSI driver: mounts secrets as files directly from the manager
   + can avoid K8s Secret objects  − files only (sync to Secret optional), rotation via remount

4. Vault Agent / sidecar injector: agent authenticates (K8s auth), renders templates to a shared volume, renews leases
   + dynamic secrets, templating  − sidecar per pod, Vault dependency

5. Encrypted in Git: SOPS (KMS-encrypted values in YAML) or Sealed Secrets (cluster-key encrypted)
   + pure GitOps  − rotation and key management harder; values (encrypted) in Git forever
```
Most common at scale: **workload identity + ESO (or CSI) + cloud secret manager**, or **Vault** where dynamic secrets/multi-cloud matter.

### Kubernetes Secret reality

K8s Secrets are **base64-encoded, not encrypted** by default. Protect them:
- **Encryption at rest in etcd** with a KMS provider (EKS envelope encryption) ([K8s 50](../../CKA-Certification-Course-2025/notes/50-encryption-basics.md)).
- **RBAC:** `get/list secrets` is powerful; few principals should have it ([K8s 57](../../CKA-Certification-Course-2025/notes/57-rbac.md)).
- Anyone who can create pods in a namespace can mount its secrets → namespace boundaries matter.

### Dynamic secrets (Vault database engine)

```
1. Pod authenticates to Vault with its K8s ServiceAccount token (Kubernetes auth method)
2. Vault policy: role "orders" may read database/creds/orders-rw
3. Vault connects to Postgres and CREATEs a user v-orders-x7f2 with grants, TTL 1 h
4. App uses it; Vault agent renews the lease while the pod lives
5. Pod dies / lease expires → Vault DROPs the user
Leak impact: one credential, traceable to one pod, valid ≤ 1 h
```

### Rotation of static secrets

- **Dual-secret (two-key) rotation** for zero downtime:
```
1. Create new secret B while A still valid (DB: create second user / second password slot; API: second key)
2. Distribute B to all consumers (they pick it up on refresh/restart)
3. Verify nothing uses A (audit logs, connection metrics)
4. Revoke A
```
- AWS Secrets Manager rotation Lambdas implement this for RDS (alternating users strategy).
- Apps must **re-read** secrets (refresh interval or reload on change) or rotation causes outages.

### KMS and encryption at rest

- Each service (S3, EBS, RDS, Secrets Manager) uses envelope encryption with a KMS key.
- **Customer-managed keys (CMK):** you control key policy (who can decrypt), rotation, and can disable the key (crypto-shredding).
- KMS calls have **rate limits** (thousands/s per region per account, varies) → cache data keys (S3 Bucket Keys reduce KMS calls).
- Key policy + IAM = two layers; cross-account access needs both.

### CI/CD secrets

- Prefer **OIDC** for cloud access (topic 32); no static cloud keys.
- Other secrets fetched at runtime from the manager using the job's identity, scoped by environment.
- Mask outputs; never `echo` secrets; avoid passing secrets via command-line args (visible in process lists).
- Scan: pre-commit hooks + CI secret scanning + platform push protection (GitHub secret scanning).

---

## Options and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| Secret manager | Cloud native vs Vault | Cloud native for single cloud and simplicity; Vault for multi-cloud/on-prem, dynamic secrets across many backends, PKI, at the cost of running a critical cluster (case study 54) |
| K8s delivery | ESO / CSI / Vault agent / SOPS | ESO for broad compatibility; CSI to avoid Secret objects; Vault agent for dynamic; SOPS for small GitOps setups |
| Static vs dynamic | Static + rotation / dynamic short-lived | Dynamic where supported (DBs, cloud creds); static with automated rotation otherwise |
| Config change path | Redeploy vs runtime config service | Redeploy for most config (safe, audited); runtime service for flags and emergency knobs |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Secret committed to Git** | Leak (public repo = scraped within minutes) | Secret scanning alerts | **Revoke/rotate immediately** (deleting the commit is not enough), then clean history; push protection |
| **Secret in logs/images/state** | Leak to wider audience | Scanners on images and logs | Scrubbing, multi-stage builds without secrets ([Docker 44](../../Docker/notes/44-handling-secrets.md)), encrypt state |
| **Rotation breaks apps** | Auth failures after rotation | Error spike at rotation time | Dual-secret rotation, apps refresh secrets, test rotation |
| **Secret manager outage / throttling** | Pods can't start (no secrets) | Fetch errors | Cache in app/K8s Secret, retries with backoff, request quotas, HA Vault |
| **Vault sealed / lost quorum** | All dynamic creds expire → mass auth failures | Vault health | HA Raft cluster across AZs, auto-unseal with KMS, DR replication (topic 54) |
| **KMS key disabled/deleted** | Data undecryptable (S3, EBS, RDS!) | Key state alarms | Restrictive key policies, deletion waiting period (7–30 days), alerts on `ScheduleKeyDeletion` |
| **Overly broad access** | One compromised pod reads all secrets | Access reviews, audit | Per-service secrets paths and roles, least privilege |
| **Bad config pushed globally** | Widespread outage | Error spike after config change | Validation, progressive rollout, instant rollback |

**3 AM:** "GitHub alert: AWS key exposed in a public commit 10 minutes ago." Immediately deactivate the key (IAM), check CloudTrail for its use, rotate any related secrets, then remove from history. Assume it was used: attackers scan public GitHub continuously.

---

## Where you'll use it

- **Case study 54:** company-wide secrets service.
- **EKS:** IRSA/Pod Identity + ESO is the standard EKS pattern; etcd envelope encryption.
- **Jenkins credentials** ([Jenkins 23–24](../../Jenkins/notes/23-credentials-model.md)); **GHA secrets/environments** ([GHA 17](../../GitHub-Actions-Basics-To-Production/notes/17-secrets-and-variables.md)).
- **Ansible Vault** vs external managers ([Ansible 40](../../Ansible/notes/40-ansible-vault.md)).
- **Terraform:** `manage_master_user_password` in RDS (Secrets Manager-managed), avoiding secrets in state.

---

## Design exercise

**Prompt:** 80 services on EKS, GitOps with Argo CD. Today secrets are K8s Secrets created by hand with `kubectl`, DB passwords are shared by several services and never rotated. Design the target.

<details>
<summary><b>Model answer</b></summary>

- **Store:** AWS Secrets Manager (single cloud) with a path convention `/{env}/{service}/{name}`, encrypted with a per-environment customer-managed KMS key.
- **Identity:** EKS Pod Identity (or IRSA): each service's ServiceAccount maps to an IAM role allowed to read only `/{env}/{service}/*`.
- **Delivery:** External Secrets Operator; each service's Helm chart includes an `ExternalSecret` (in Git, no values) → K8s Secret, refresh every 1 h. etcd envelope encryption on; RBAC denies `get secrets` to humans in prod except break-glass.
- **Databases:** one DB user per service (least privilege); passwords managed by Secrets Manager rotation (alternating-users strategy) every 30 days; apps reload credentials on auth failure / via file watch (or use RDS IAM authentication tokens, valid 15 min, where supported). If we adopt Vault later: dynamic DB creds.
- **Migration:** inventory existing Secrets → create entries in Secrets Manager → add ExternalSecrets service by service → split shared DB users → rotate everything (assume old values leaked) → delete hand-made Secrets.
- **CI:** OIDC only; secret scanning with push protection; pre-commit gitleaks.
- **Monitoring:** CloudTrail alerts on secret access by unexpected principals, ESO sync failures, rotation failures, KMS key deletion attempts.
</details>

---

## Quick recap

- Config = behaviour (review and roll out like code); secrets = access (secret manager + KMS).
- Solve secret zero with workload identity; avoid static cloud keys entirely (OIDC, IRSA/Pod Identity).
- Prefer dynamic, short-lived credentials; rotate static ones with dual-secret rotation and apps that refresh.
- K8s: ESO/CSI/Vault agent; encrypt etcd; restrict `get secrets`; SOPS/Sealed Secrets for pure GitOps.
- A leaked secret must be revoked, not just deleted from Git; secret managers and KMS are critical dependencies to make HA.

### Interview questions

1. **Q: How do pods on EKS get database credentials securely?**
   A: The pod's ServiceAccount is bound to an IAM role (Pod Identity/IRSA). An operator like External Secrets (or the app SDK, or the CSI driver) uses that identity to read only that service's secret from Secrets Manager, encrypted with KMS, into a K8s Secret or mounted file. Better still: dynamic credentials (Vault DB engine) or RDS IAM auth tokens that expire in minutes. etcd encryption and RBAC protect the in-cluster copy.

2. **Q: What is envelope encryption and why use it?**
   A: Data is encrypted locally with a data key; the data key is encrypted by a master key held in KMS. Only the small data key goes to KMS for decryption, so it's fast and scalable, the master key never leaves the HSM, access is controlled and audited by KMS policy, and master key rotation doesn't require re-encrypting all data.

3. **Q: How do you rotate a database password without downtime?**
   A: Two credentials overlap: create the new one while the old still works, roll it out to all consumers (who reload secrets), verify the old is no longer used, then revoke it. Managed rotation often alternates between two DB users. Apps must refresh secrets, and the procedure should be tested regularly, not invented during an incident.
