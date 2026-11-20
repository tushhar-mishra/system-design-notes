# 40 · Security Architecture

> **Level:** ★★★ advanced
>
> **You will learn:** how to design security into a system: **threat modelling** (STRIDE), **defence in depth**, **zero trust**, **identity and access** (IAM least privilege, roles over users, SSO, break-glass), **network segmentation** and **egress control**, **encryption** in transit and at rest, **edge protection** (WAF, DDoS, bot management), **workload and container security**, **software supply chain** (SLSA, signing, SBOM), **detection and audit** (CloudTrail, GuardDuty, SIEM), **vulnerability management**, compliance basics, and the failures that cause breaches.
>
> **You need:** topics 05, 08, 10, 15, 25, 32, 36–38.
>
> **Part 8 · Security, DR and cost** · Previous: [39 · Platform engineering](39-platform-engineering.md) · Next: [41 · Backups and DR](41-backups-and-disaster-recovery.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Security architecture assumes **something will be compromised** and limits what an attacker can do next. Layers:

```
Edge:      WAF, DDoS protection, rate limits, bot rules                    (topics 15, 25)
Identity:  SSO + MFA for humans, workload identity for machines, least privilege, no long-lived keys
Network:   private subnets, security groups by reference, network policies, egress allow-lists
Data:      encryption in transit (TLS/mTLS) and at rest (KMS), backups immutable
Workload:  minimal signed images, non-root, read-only FS, policies at admission, runtime detection
Supply chain: protected source, isolated builds, SBOM, signatures, provenance
Detect & respond: audit logs centralized and immutable, threat detection, alerts, IR runbooks
```

**Picture it.** A bank. Guards at the door (WAF), ID badges for staff (IAM), different keys for different rooms (least privilege), a vault inside a locked room inside the building (defence in depth), cameras everywhere (audit logs), and alarms (detection). No single lock protects the money.

**The one thing to know.** **Identity is the new perimeter.** Most cloud breaches come from **stolen or over-privileged credentials** and **misconfigurations** (public buckets, open security groups), not exotic exploits. Short-lived credentials + least privilege + guardrails prevent most of them.

**Try it now:** `aws iam generate-credential-report` then look for old access keys and users without MFA. Or in K8s: `kubectl auth can-i --list --as system:serviceaccount:default:default`.

**You've got it if you can:** run a quick STRIDE threat model, design least-privilege IAM for a service, and list the layers protecting a web app.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Concept | Meaning |
|---|---|
| **Threat model** | Structured look at what you're protecting, from whom, and how it can be attacked |
| **STRIDE** | Threat categories: **S**poofing (identity), **T**ampering (data), **R**epudiation (deny actions), **I**nformation disclosure, **D**enial of service, **E**levation of privilege |
| **Defence in depth** | Multiple independent layers; one failing doesn't expose everything |
| **Zero trust** | Never trust by network location; authenticate and authorize every request based on identity, device and context |
| **Least privilege** | Every identity gets only the permissions it needs, for only as long as needed |
| **Blast radius** | What an attacker can reach from one compromised component |
| **Shared responsibility model** | Cloud provider secures the cloud (hardware, hypervisor, managed service internals); you secure what you put in it (IAM, configs, data, code) |
| **SIEM** | Security Information and Event Management: central log analysis for security (Splunk, Sentinel, Chronicle, OpenSearch Security Analytics) |

---

## Why it exists

Breaches cost money, customer trust, regulatory fines (GDPR up to 4% of global turnover) and sometimes the company. DevOps/platform engineers control the most powerful credentials (CI, IaC, clusters), so security design is their job, not an afterthought. Interviews for senior roles include a security section in every design.

---

## Analogy

The bank (Core). **Zero trust** = even staff inside the building swipe their badge at every door, and the system checks whether *this* person should be in *this* room *now*.

---

## How it works inside

### Threat modelling in 15 minutes (for any design)

```
1. Draw the data flow: users → edge → services → data stores → third parties; mark trust boundaries
2. List assets: customer PII, payment data, credentials, source code, CI signing keys
3. Walk each boundary with STRIDE:
   - Spoofing: can someone pretend to be a user/service? → authN, mTLS
   - Tampering: can data/artifacts be modified? → integrity checks, signing, IAM
   - Repudiation: can someone deny doing it? → audit logs, immutable
   - Information disclosure: leaks? → encryption, access control, no secrets in logs
   - DoS: can it be overwhelmed? → rate limits, WAF, autoscaling, quotas
   - Elevation of privilege: can a low-priv actor gain more? → least privilege, isolation, patching
4. Rank by likelihood × impact; mitigate top risks; record accepted risks
```

### Identity and access

**Humans:**
- **SSO** (Single Sign-On) via an IdP (Okta, Entra ID, Google) with **MFA**; cloud access through IAM Identity Center / workforce identity federation; **no IAM users with passwords/keys**.
- **Role-based**, time-bound elevated access (just-in-time: request prod write access for 1 hour with approval).
- **Break-glass** accounts: sealed, MFA hardware keys, alert on any use.
- Root account: MFA, no keys, unused.

**Machines:**
- **Workload identity** everywhere (IRSA/Pod Identity, instance profiles, OIDC from CI) → short-lived credentials (topic 37).
- One role per service per environment, scoped to its resources (ARNs), not `*`.
- **Permission boundaries** and **SCPs** cap what any role in an account can do (guardrails, topic 36).
- Review with access analyzers (IAM Access Analyzer unused access findings) and remove unused permissions.

```json
{ "Effect": "Allow",
  "Action": ["sqs:SendMessage"],
  "Resource": "arn:aws:sqs:ap-south-1:111122223333:prod-orders-events" }
```
vs ⚠️ `"Action": "sqs:*", "Resource": "*"`.

**Kubernetes:** RBAC by namespace, no cluster-admin for humans by default, ServiceAccount per app, `automountServiceAccountToken: false` where not needed, audit logging on the API server ([K8s 54–57](../../CKA-Certification-Course-2025/notes/56-authorization-modes.md)).

### Network segmentation and egress

```
Internet ─▶ [CDN + WAF] ─▶ [ALB in public subnets] ─▶ [apps in private subnets] ─▶ [data in isolated subnets]
SGs: alb-sg → app-sg → db-sg (by reference) ; NACLs as coarse guardrails
K8s: default-deny NetworkPolicies, allow only declared callers ; mesh authorization by identity
Egress: NAT + egress firewall / proxy with domain allow-list (stops data exfiltration & malware callbacks)
Admin access: no SSH from internet; SSM Session Manager / bastion with SSO + session logging
Private endpoints for cloud APIs (topic 08)
```

### Encryption

- **In transit:** TLS everywhere at the edge; re-encrypt or mTLS internally for sensitive data (topic 05).
- **At rest:** KMS-backed encryption on all storage (default on most services now); customer-managed keys for sensitive data with restrictive key policies (topic 37).
- **Application-level / field-level encryption** or **tokenization** for the most sensitive fields (card numbers, national IDs), so DB admins and backups don't see plaintext.

### Edge protection

- **DDoS:** L3/L4 handled by cloud/CDN (AWS Shield Standard is automatic; Shield Advanced adds response team + cost protection; GCP Cloud Armor; Azure DDoS Protection). L7 floods need WAF rate rules, caching and autoscaling.
- **WAF** (Web Application Firewall): managed rule sets (OWASP Top 10 patterns: SQL injection, XSS), IP reputation, rate-based rules, geo rules, bot control. Start in **count mode**, then block.
- **Origin lock-down:** only the CDN can reach the origin (topic 15).

### Workload and container security

| Layer | Controls |
|---|---|
| Image | Minimal/distroless base, no secrets baked in, scanned (Trivy/Grype), rebuilt regularly, signed ([Docker 43](../../Docker/notes/43-image-supply-chain.md)) |
| Pod | Non-root, read-only root FS, drop all capabilities, no privilege escalation, seccomp `RuntimeDefault` ([Docker 41](../../Docker/notes/41-users-and-capabilities.md), [K8s 59](../../CKA-Certification-Course-2025/notes/59-securitycontext-and-capabilities.md)) |
| Admission | Pod Security `restricted`, Kyverno/Gatekeeper: signed images only, registry allow-list, no `hostPath`/`hostNetwork` |
| Runtime | Falco/GuardDuty runtime monitoring: shell in container, unexpected outbound, crypto-miners |
| Node | Hardened minimal OS (Bottlerocket, Container-Optimized OS), automatic patching via node replacement, IMDSv2 required with hop limit 1 (stops pods stealing node credentials) |

### Software supply chain

From topic 32: protected branches + reviews → isolated ephemeral builds → SBOM → signatures + provenance (SLSA) → registry scanning → admission verification. Plus: dependency pinning and update bots, private package proxies (prevent dependency confusion), pinned CI actions, separate signing identities per environment.

### Detection, audit, response

```
CloudTrail (org trail, all regions) ─┐
VPC Flow Logs, DNS query logs ───────┼─▶ log-archive account (S3 Object Lock, separate access) ─▶ SIEM
K8s audit logs, app auth logs ───────┘
GuardDuty / Security Command Center / Defender ─▶ findings ─▶ Security Hub ─▶ alerts/tickets
AWS Config / policy scanners ─▶ misconfiguration findings (public bucket, open SG) ─▶ auto-remediation
```
- Logs in a **separate account** so a compromised workload account can't delete them.
- **IR runbooks:** credential leak, compromised pod, public bucket, ransomware (topic 30 process + security specifics: preserve evidence, isolate, rotate).

### Vulnerability management

- Scan: code (SAST), dependencies (SCA), images, IaC (Checkov/Trivy), running workloads, hosts.
- **Prioritize** by exploitability and exposure (CVSS + EPSS + "is it reachable / internet-facing?"), not by count.
- SLAs: e.g. critical internet-facing within 7 days.
- Patch by **rebuilding and redeploying** (immutable infra), not by SSH patching.

### Compliance (in one table)

| Framework | What it cares about |
|---|---|
| **SOC 2** | Controls for security/availability/confidentiality, evidence of operation |
| **ISO 27001** | Information security management system |
| **PCI DSS** | Card data: segmentation, encryption, access, logging, scanning |
| **HIPAA** (US health), **GDPR** (EU privacy), **DPDP Act** (India data protection) | Personal data handling, consent, residency, breach notification |

Design lever: **reduce scope** (isolate the regulated part into its own accounts/clusters/network), automate evidence (IaC, policy reports, audit logs).

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Perimeter vs zero trust | VPN + trusted network: simple, broad blast radius | Identity-aware access per request (zero trust): more setup, much smaller blast radius |
| Policy enforcement | Detective (find and alert) | Preventive (block at admission/SCP): safer, can block legit work; use both |
| WAF mode | Count (observe) | Block (protect, risk false positives) → start count, then block |
| Key management | Cloud-managed keys (simple) | Customer-managed (control, audit, crypto-shredding; more ops) |
| Security tooling | Cloud-native (GuardDuty, Security Hub) | Third-party CNAPP/SIEM (multi-cloud, cost) |

Security vs velocity: put controls **in the platform** (templates, admission, SCPs) so they're automatic, rather than in manual review gates.

---

## Failure modes (common breach paths)

| Failure | Example | Detect | Prevent |
|---|---|---|---|
| **Leaked long-lived keys** | AWS keys in GitHub, CI logs, laptops | Secret scanning, CloudTrail anomalies | No long-lived keys: SSO, OIDC, workload identity |
| **Over-privileged role** | App role with `s3:*` on `*` used to exfiltrate all buckets | Access Analyzer | Least privilege, permission boundaries |
| **Public data store** | Open S3 bucket / Elasticsearch / DB snapshot shared publicly | Config rules, Macie | Block Public Access (account-wide), SCPs, policy as code |
| **SSRF to metadata service** | App fetches attacker URL `http://169.254.169.254/...` → node credentials | GuardDuty | IMDSv2 with hop limit 1, egress filtering, workload identity |
| **Container escape / privileged pod** | `privileged: true` pod → node root → all pods' secrets | Runtime detection | Pod Security restricted, admission policies |
| **CI compromise** | Malicious PR or third-party action steals deploy creds | Audit, anomaly alerts | Isolated runners, OIDC scoped to protected branches, pinned actions (topic 32) |
| **Unpatched internet-facing service** | Known CVE exploited | Scanners, WAF alerts | Patch SLAs, WAF virtual patching, minimal exposure |
| **Logs deleted by attacker** | Cover tracks | Gaps in logs | Separate log-archive account, Object Lock |
| **No MFA on admin** | Phished password → full access | IdP reports | Phishing-resistant MFA (FIDO2 keys) |

**3 AM:** "GuardDuty: EC2 node role credentials used from an external IP." Treat as compromise: revoke sessions for that role (IAM "revoke active sessions"), isolate the node (SG quarantine, keep for forensics), find the entry point (SSRF? malicious image?), rotate secrets reachable from it, check CloudTrail for what the attacker did, follow the IR process and notification obligations.

---

## Where you'll use it

- Every design's **security section** in interviews (case studies).
- **Kubernetes security** chapters ([K8s 54–63](../../CKA-Certification-Course-2025/notes/58-admission-control.md)), **Docker security** ([Docker 40](../../Docker/notes/40-security-model.md)), **Linux hardening** ([Linux 066](../../linux-mastery-rhel/notes/066-ssh-server-hardening.md), [Linux 072 · SELinux](../../linux-mastery-rhel/notes/072-selinux-fundamentals.md)), **Ansible** ([Ansible 60](../../Ansible/notes/60-security.md)), **Jenkins** ([Jenkins 65](../../Jenkins/notes/65-securing-jenkins.md)), **Terraform scanning** ([Terraform 53](../../Terraform/notes/53-linting-and-scanning.md)).

---

## Design exercise

**Prompt:** Threat-model and secure a public SaaS API on EKS that stores customer PII (names, phone numbers) in Aurora, uses S3 for document uploads, and has a GitHub Actions CI/CD pipeline. List the top 6 threats and controls.

<details>
<summary><b>Model answer</b></summary>

| # | Threat (STRIDE) | Control |
|---|---|---|
| 1 | Credential theft → cloud takeover (S, E) | SSO + FIDO2 MFA for humans, no IAM users/keys, OIDC for CI scoped to `main` + environment approvals, IRSA/Pod Identity per service, SCPs (deny disabling CloudTrail/GuardDuty, deny leaving regions), break-glass with alerts |
| 2 | PII disclosure from S3/Aurora (I) | Private buckets with Block Public Access at account level, bucket policies limited to service roles + VPC endpoint, SSE-KMS with CMK, Aurora encrypted + IAM auth or rotated secrets, field-level encryption for phone numbers, Macie scans |
| 3 | Malicious uploads / SSRF (T, E) | Pre-signed upload URLs with size/type limits, async malware scanning before use, IMDSv2 hop limit 1, egress allow-list |
| 4 | Supply-chain compromise via CI (T, E) | Isolated runners for PRs, pinned actions, SBOM + cosign signing + provenance, Kyverno verifies signatures at admission, branch protection with reviews |
| 5 | L7 abuse / DoS / credential stuffing (D, S) | CloudFront + WAF managed rules + rate rules, per-key rate limits at gateway, login throttling, bot control |
| 6 | Undetected intrusion / repudiation (R) | Org CloudTrail + K8s audit + VPC flow logs into a log-archive account with Object Lock, GuardDuty (incl. EKS runtime), Security Hub, alerts with IR runbooks |

Plus: default-deny NetworkPolicies, Pod Security restricted, mTLS between services if compliance requires, vulnerability SLAs, quarterly access reviews, a DPDP/GDPR data map (where PII lives, retention, deletion process).
</details>

---

## Quick recap

- Threat-model with STRIDE at trust boundaries; defence in depth; zero trust.
- Identity is the perimeter: SSO+MFA, workload identity, short-lived creds, least privilege, SCP guardrails.
- Segment networks, control egress, encrypt in transit and at rest, protect the edge (WAF/DDoS).
- Secure workloads (minimal signed images, restricted pods, admission policies, runtime detection) and the supply chain.
- Centralize immutable audit logs, detect threats, have IR runbooks; reduce compliance scope by isolation.

### Interview questions

1. **Q: What are the most common causes of cloud breaches and how do you prevent them?**
   A: Stolen or leaked credentials, over-privileged identities, and misconfigurations like public buckets or open security groups. Prevent with SSO+MFA, no long-lived keys (OIDC, workload identity), least privilege with boundaries and SCPs, preventive guardrails (Block Public Access, policy as code), and continuous detection (Config, GuardDuty) with centralized audit logs.

2. **Q: How would you apply zero trust to service-to-service traffic?**
   A: Every service gets a cryptographic identity (mTLS certs from a mesh or SPIFFE), every call is authenticated and authorized by identity via policy (only orders may call payments), default-deny network policies as a second layer, short-lived credentials, and audit of calls. Network location alone grants nothing.

3. **Q: A pod is compromised. What limits the damage in a well-designed platform?**
   A: Non-root, no capabilities, read-only FS and seccomp make escape hard; its ServiceAccount maps to a narrowly scoped IAM role; NetworkPolicies limit what it can reach; egress allow-lists block exfiltration; IMDSv2 hop limit stops node-credential theft; secrets are per-service; runtime detection alerts on shells or odd connections; and audit logs show what happened.
