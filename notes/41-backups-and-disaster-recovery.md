# 41 · Backups and Disaster Recovery

> **Level:** ★★★ advanced
>
> **You will learn:** **RPO** (Recovery Point Objective) and **RTO** (Recovery Time Objective) and how to derive them from business impact, what counts as a **disaster** (region loss, data corruption, ransomware, account compromise, human error), **backup types** (full, incremental, snapshots, PITR, logical vs physical), the **3-2-1-1-0 rule**, **immutable and isolated backups**, the four **DR strategies** (**backup & restore, pilot light, warm standby, multi-site active-active**) with cost and RTO/RPO, **DR for Kubernetes and stateful platforms**, **runbooks**, and **testing restores** (the part everyone skips).
>
> **You need:** topics 17, 20, 23, 36–38.
>
> **Part 8 · Security, DR and cost** · Previous: [40 · Security](40-security-architecture.md) · Next: [42 · Multi-region architecture](42-multi-region-architecture.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.**
```
RPO = how much DATA you can lose, measured in time   ("we can lose at most the last 5 minutes of orders")
RTO = how long you can be DOWN                       ("we must be back within 1 hour")

    last good copy                 disaster                          service restored
 ────────●──────────────────────────✖────────────────────────────────────●────────▶ time
         └────────── RPO ───────────┘└──────────────── RTO ───────────────┘
```

**DR strategies (cheap/slow → expensive/fast):**
```
Backup & restore   RPO hours      RTO hours–days   cost  ×1.1      rebuild everything from backups + IaC
Pilot light        RPO minutes    RTO tens of min–hours  ×1.2–1.4  data replicated, core infra off/minimal
Warm standby       RPO seconds–min RTO minutes    ×1.5–2    scaled-down full copy running, scale up on failover
Active-active      RPO ~0–seconds RTO ~0–minutes  ×2+       both regions serve traffic (topic 42)
```

**Picture it.** Home insurance against a fire. Backup & restore = photos of your belongings stored at a friend's house (you'll rebuild, slowly). Pilot light = a second, empty flat with furniture in storage nearby. Warm standby = a furnished second flat you can move into today. Active-active = living in two homes at once.

**The one thing to know.** **A backup you haven't restored is a hope, not a backup.** Test restores regularly, measure the real RTO, and keep at least one copy **immutable and in a separate account**, because ransomware and compromised admins delete backups first.

**Try it now:** pick one database you run. When was its last *restore* test? How long did it take? That's your real RTO.

**You've got it if you can:** derive RPO/RTO from business needs, pick a DR strategy with cost reasoning, and write the outline of a DR runbook.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Meaning |
|---|---|
| **Disaster** | An event that takes out a system beyond normal redundancy: region outage, data corruption, deleted resources, ransomware, account takeover, provider failure |
| **HA vs DR** | **High availability** handles component/AZ failures automatically within a region (topic 23). **DR** recovers from bigger or logical disasters, often across regions/accounts, often with a decision by humans. |
| **Backup** | A copy of data from a point in time, stored separately |
| **Snapshot** | Point-in-time copy of a volume/DB (incremental at block level in clouds) |
| **PITR** (Point-In-Time Recovery) | Restore to any second within a window using base backups + transaction logs (WAL/binlog) |
| **Logical backup** | Export of data as SQL/records (`pg_dump`): portable, slow for big DBs |
| **Physical backup** | Copy of data files/blocks: fast, version-specific |
| **3-2-1-1-0 rule** | **3** copies, **2** different media/storage types, **1** off-site (other region/account), **1** immutable/offline, **0** errors in restore verification |
| **Immutable backup** | Can't be modified or deleted before retention ends (S3 Object Lock compliance mode, AWS Backup Vault Lock) |
| **Failover / failback** | Moving to the DR site / moving back afterwards |

---

## Why it exists

Replication protects against hardware failure but **copies mistakes and attacks instantly** (topic 17): a bad migration, a `DELETE` without `WHERE`, ransomware encrypting data, an attacker deleting your account's resources. Regions do have large outages. Regulators and customers ask for proven DR. And RTO/RPO decide architecture and cost, so they must be explicit.

---

## Analogy

Home insurance tiers (Core). **Testing restores** = actually trying to move into the spare flat once a year, to discover the key doesn't fit before the fire, not during it.

---

## How it works inside

### Deriving RPO/RTO (business impact analysis)

```
For each system ask the business:
- What does 1 hour of downtime cost? (revenue, SLA penalties, reputation, safety, legal)
- What does losing 1 hour of data cost? Can it be re-created (re-entered, re-imported, replayed from a queue)?
- What's the regulatory requirement?
Then tier systems:
Tier 0 (payments, auth):          RPO ≈ 0–1 min,   RTO < 15–60 min
Tier 1 (core customer features):  RPO < 15 min,    RTO < 4 h
Tier 2 (internal tools):          RPO < 24 h,      RTO < 1–2 days
Tier 3 (analytics, dev):          RPO < 1 week,    RTO best effort
```
Each tier's RTO/RPO determines the DR strategy and budget. Don't give everything tier 0.

### Backup design

| Data | Method | Notes |
|---|---|---|
| Managed relational DB | Automated backups + **PITR** (e.g. RDS up to 35 days), plus snapshots copied to another region **and** another account | Restore creates a **new** instance (new endpoint) → apps must be repointed |
| Self-managed Postgres | pgBackRest/WAL-G: base backups + continuous WAL archiving to S3 → PITR | Test restore weekly |
| DynamoDB | PITR (35 days) + on-demand backups, AWS Backup cross-account copy | |
| Object storage | Versioning + replication to another region/account + Object Lock | Lifecycle for old versions |
| Block volumes | Snapshots (AWS Backup / Data Lifecycle Manager) | Crash-consistent unless app-coordinated |
| Kubernetes | GitOps repo (desired state) + **Velero** (K8s objects + PV snapshots) + etcd snapshots if self-managed | Most K8s state should be rebuildable from Git |
| Secrets | Secret manager replication (multi-region secrets), Vault snapshots | Without secrets you can't restore anything |
| IaC state | Versioned S3 state bucket, replicated | Needed to rebuild infra |
| Queues/streams | Usually not backed up; replay from source or MirrorMaker/cross-region replication for Kafka | Decide explicitly |
| Config, CI, registries | Git (mirrored), registry replication | Your **recovery tooling** must survive too |

**Immutable + isolated:** use a dedicated **backup account** with its own credentials, Vault Lock / Object Lock (compliance mode = even root can't delete before retention ends), and no standing access from workload accounts.

### The four DR strategies in detail

```
1. BACKUP & RESTORE
   Primary region: everything.  DR region: backups only (+ IaC ready).
   Failover: run Terraform in DR region → restore DBs from latest cross-region backups → deploy apps (GitOps) → switch DNS.
   RPO: backup frequency (hours, or minutes with PITR log shipping). RTO: hours (infra build + data restore size).

2. PILOT LIGHT
   DR region: data continuously replicated (cross-region read replica / Aurora Global DB / S3 CRR),
              core infra exists (VPC, maybe EKS control plane), app capacity at ZERO or minimal.
   Failover: promote DB replica → scale up/deploy apps → switch traffic.
   RPO: replication lag (seconds–minutes). RTO: tens of minutes–hours.

3. WARM STANDBY
   DR region: full stack running at reduced capacity (e.g. 10–30%), data replicated, tested continuously.
   Failover: promote DB, scale out (autoscaling + pre-raised quotas), switch traffic.
   RPO: seconds. RTO: minutes.

4. MULTI-SITE ACTIVE-ACTIVE
   Both regions serve live traffic; data replicated both ways or partitioned by user (topic 42).
   Failover: stop sending traffic to the failed region (automatic health-based routing).
   RPO: ~0–seconds (depends on replication). RTO: ~minutes or less. Cost and complexity highest.
```

| Strategy | RPO | RTO | Cost | Complexity |
|---|---|---|---|---|
| Backup & restore | Hours (minutes with log shipping) | Hours–day | Lowest | Low (but restore at scale is hard) |
| Pilot light | Seconds–minutes | 30 min–few hours | Low–medium | Medium |
| Warm standby | Seconds | Minutes | Medium–high | Medium–high |
| Active-active | ~0 | ~0–minutes | Highest | Highest (data consistency) |

### What else must be in the DR region (often forgotten)

- **Quotas** raised in advance (vCPUs, IPs, LB counts).
- **Container images and artifacts** replicated (ECR replication).
- **Secrets and KMS keys** (multi-region keys / replicated secrets).
- **DNS** and certificates for DR endpoints.
- **Third-party allow-lists** (partners who whitelist your egress IPs need DR IPs too).
- **Observability** in the DR region (otherwise you fail over blind).
- **CI/CD and GitOps** able to deploy there (if the CI lives only in the dead region...).
- **People:** runbooks, access, and who decides.

### DR runbook outline

```
1. Trigger & decision: criteria for declaring DR (e.g. region impaired > 30 min with no ETA), who decides (IC + exec)
2. Communicate: incident channel, status page, stakeholders
3. Freeze writes in primary if reachable (avoid split brain), record last replicated position
4. Promote data stores in DR (order matters: DB → caches warm → queues)
5. Scale/deploy compute in DR (GitOps sync, autoscaling minimums raised)
6. Verify with smoke tests and SLIs
7. Switch traffic (DNS failover / global LB) ; monitor
8. Data reconciliation: identify transactions possibly lost (RPO window), replay from queues/logs if possible
9. Failback plan (later, planned): re-sync primary from DR, switch back in a maintenance window
```

### Testing (the important part)

| Test | Frequency | What it proves |
|---|---|---|
| Automated restore test of each backup type (restore to a scratch environment, run checks) | Weekly/daily | Backups are restorable; measures restore time |
| Tabletop exercise (walk through the runbook on paper) | Quarterly | People know the steps; gaps in the plan |
| Partial failover (one service, or DR region with synthetic traffic) | Quarterly | Infra and automation work |
| Full DR game day (real failover of production, or of a production-like environment) | 1–2× per year | Real RTO/RPO, surprises (topic 44) |

Record the measured RTO/RPO; if they miss the target, that's a reliability bug.

---

## Options and trade-offs

| Decision | Option A | Option B |
|---|---|---|
| DR strategy per tier | Same strategy for all (simple, expensive or inadequate) | Tiered strategies ✅ |
| Backup location | Same account (convenient) | Separate locked account ✅ (survives compromise) |
| Restore approach | Restore in place | Restore to new resources and switch ✅ (keeps evidence, safer) |
| Automatic vs manual regional failover | Automatic (fast, risk of false failover and split brain) | Human decision with automated steps ✅ for most (seconds don't matter as much as correctness) |
| Kubernetes DR | Velero backup/restore of clusters | Rebuild clusters from IaC + GitOps, data from data-layer DR ✅ (clusters as cattle) |

---

## Failure modes

| Failure | Symptom | Prevent |
|---|---|---|
| **Backups never tested** | Restore fails or takes 30 h when needed | Automated restore tests with checks and timing |
| **Backups deleted by attacker / mistake** | Nothing to restore | Immutable, separate account, MFA delete, Vault Lock |
| **Backup consistent per system but not across systems** | DB restored to 10:00, S3 to 09:00, queue offsets to 10:05 → inconsistent | Know dependencies; reconciliation procedures; idempotent replays |
| **DR region missing quotas/images/secrets** | Failover stalls | DR readiness checks in CI; regular partial failovers |
| **Restore time grows with data** | RTO silently becomes days | Track restore duration trend; incremental strategies; warm standby for big DBs |
| **Split brain on failover** | Both regions accept writes | Fence primary (stop writes) before promotion; single decision maker |
| **Failback never planned** | Stuck in DR region indefinitely | Documented failback, tested |
| **Encryption keys not available** | Backups unreadable | Multi-region keys / key copies; key policy covers DR account |
| **Recovery tooling in the same blast radius** | CI, Git, runbooks unreachable during disaster | Mirror Git, offline runbook copy, DR-capable CI path |

**3 AM:** "Someone ran a migration that corrupted the orders table 40 minutes ago." This is a **logical** disaster: replicas already have the corruption. Use **PITR** to restore to a new instance at the time just before the migration, extract/compare the affected rows, and repair the production table (or switch to the restored instance if simpler and acceptable), while freezing further writes to the affected data if possible.

---

## Where you'll use it

- **Case study 52** (DR for a critical database) and **51** (multi-region K8s).
- **etcd backup/restore** ([K8s 77](../../CKA-Certification-Course-2025/notes/77-etcd-backup-restore-and-dr.md)), **Jenkins backup** ([Jenkins 68](../../Jenkins/notes/68-backup-restore-upgrades.md)), **Terraform state recovery** ([Terraform 42](../../Terraform/notes/42-state-recovery.md)), **LVM snapshots and rsync backups on Linux** ([Linux 052](../../linux-mastery-rhel/notes/052-lvm-snapshots-and-recovery.md), [Linux 068](../../linux-mastery-rhel/notes/068-rsync-and-backups.md)).
- **Terraform:** `aws_backup_plan`, `aws_backup_vault_lock_configuration`, cross-region copy rules.

---

## Design exercise

**Prompt:** An e-commerce platform in `ap-south-1`: EKS, Aurora PostgreSQL (2 TB), S3 product images (40 TB), Redis cache, SQS queues. Business says: orders RPO 5 min / RTO 1 h for a region loss; product images RPO 24 h / RTO 4 h; ransomware must not destroy backups. Budget is limited. Design DR.

<details>
<summary><b>Model answer</b></summary>

- **Strategy:** **pilot light** in `ap-southeast-1` (or another Indian region if data residency requires, e.g. `ap-south-2` Hyderabad): meets RTO 1 h at far lower cost than warm standby, if automation is solid.
- **Orders DB:** Aurora Global Database secondary cluster in DR (async, typically < 1 s lag → RPO well under 5 min), minimal instance size; promotion via managed failover (minutes). Plus PITR (35 days) and daily snapshots copied to a **separate backup account** in a third location with **AWS Backup Vault Lock (compliance mode, 35-day retention)** for ransomware/logical disasters.
- **S3 images:** Cross-Region Replication to the DR bucket (RPO minutes, better than required) + versioning + Object Lock (governance) on the backup-account copy. 40 TB × replica storage is the main extra cost → DR bucket in a cheaper class (Standard-IA) is acceptable given RTO 4 h.
- **Redis:** not backed up; rebuilt empty in DR and warmed (cache is optional by design, topic 14).
- **SQS:** in-flight messages in the dead region are lost/stuck → producers use outbox (topic 22) so events are re-published from the DB after failover.
- **EKS:** DR cluster defined in Terraform; pilot light = cluster + system node pool running, app node pools at 0; Argo CD in DR syncs the same GitOps repo (DR values: lower min replicas). ECR replication; Secrets Manager multi-region secrets; multi-region KMS keys; quotas pre-raised.
- **Traffic:** Route 53 failover records (or Global Accelerator) to the DR ALB; certificates in ACM in both regions.
- **Runbook + automation:** one pipeline "Failover to DR" (promote Aurora, scale node pools, raise HPA minimums, run smoke tests, flip DNS). Target: 45 minutes.
- **Testing:** weekly automated restore of the latest snapshot to a scratch instance with row-count checks; quarterly DR drill in a staging copy; yearly production game day.
- **Cost estimate (relative):** ~+20–30% of the production bill (Aurora secondary, replicated storage, idle cluster), vs ~+80–100% for warm standby.
</details>

---

## Quick recap

- RPO = data loss tolerance; RTO = downtime tolerance; derive them per system tier from business impact.
- Replication ≠ backup: keep PITR + snapshots, immutable, in a separate account (3-2-1-1-0).
- Strategies: backup & restore → pilot light → warm standby → active-active, trading cost for RTO/RPO.
- DR region needs data, images, secrets, keys, quotas, DNS, observability, CI path and runbooks.
- Test restores and failovers regularly; measured RTO/RPO is the only real one.

### Interview questions

1. **Q: What's the difference between RPO and RTO? Give an example where they drive different designs.**
   A: RPO is how much data loss is acceptable; RTO is how long recovery may take. A reporting system might accept RPO 24 h (rebuild from source) but need RTO 1 h (executives need it) → keep a warm environment, restore from yesterday's export. A ledger needs RPO ≈ 0 but might tolerate RTO 4 h → synchronous replication and immutable logs matter more than instant failover.

2. **Q: How do you protect backups from ransomware?**
   A: Store copies in a separate account with separate credentials, use immutability (S3 Object Lock compliance mode, Backup Vault Lock), restrict and alert on deletion attempts, keep multiple retention points (to restore from before the infection), encrypt with keys the attacker can't delete, and regularly test restores, including from the isolated copies.

3. **Q: Pilot light vs warm standby: how do you choose?**
   A: From RTO and budget. Pilot light keeps data replicated and core infrastructure ready but compute off, giving RTO in tens of minutes to hours at low cost. Warm standby runs a scaled-down full stack, giving RTO in minutes and continuous validation, at much higher cost. If the business needs minutes, or failover automation isn't mature enough to scale from zero reliably, choose warm standby.
