# 52 · Case Study: DR for a Critical Database

> **Level:** ★★★ advanced
>
> **You will practise:** designing **disaster recovery for one critical database** against a given **RPO/RTO**: classifying disasters (hardware, AZ, region, **logical corruption**, **ransomware/account compromise**), combining **HA replication + PITR + immutable cross-account backups**, **failover automation and fencing**, **application reconnection**, **restore testing**, **runbooks**, and **cost**.
>
> **You need:** topics 16–17, 26, 35, 40–42.
>
> **Part 9 · Case studies** · Previous: [51 · Multi-region Kubernetes](51-case-multi-region-kubernetes.md) · Next: [53 · Self-service infrastructure](53-case-self-service-infrastructure.md) · [Back to INDEX](../INDEX.md)

---

## 1. Requirements and clarifying questions

**Prompt:** "Design DR for our core payments PostgreSQL database."

| Question | Assumed answer |
|---|---|
| Size and load? | 4 TB, 3,000 writes/s, 20,000 reads/s; grows 1 TB/year |
| Current setup? | RDS PostgreSQL Multi-AZ, daily snapshots, 7-day PITR |
| RPO/RTO by disaster type? | Instance/AZ: RPO 0, RTO 2 min. Region: RPO ≤ 1 min, RTO ≤ 30 min. Logical corruption (bad deploy/migration, operator error): recover data to any point in last 35 days within 4 h. Ransomware/account compromise: recover within 24 h from backups the attacker can't touch |
| Compliance? | Payments: 7-year retention of financial records; audit of access |
| Dependencies? | 12 services; outbox events to Kafka; reporting via CDC to a warehouse |
| Budget? | Payments is tier 0; budget available but must be justified |

---

## 2. Estimation

```
WAL generation: 3,000 writes/s × ~1–2 KB per write ≈ 3–6 MB/s → ~260–520 GB/day of WAL
Cross-region replication bandwidth: ~5 MB/s → fine; transfer ≈ 13 TB/month × $0.02 ≈ $260/month (illustrative)
Snapshot restore time: managed snapshot restores are lazy-loaded from S3 (instance available in ~tens of minutes,
   but first reads are slow until blocks are hydrated) → for 4 TB, full performance may take hours → impacts the logical-corruption RTO
PITR: restore = snapshot + WAL replay up to target time → plus WAL replay time (could be ~1 h for a day of WAL)
Logical backup (pg_dump) of 4 TB: many hours → only for long-term archives / table-level exports, not RTO
```
**Conclusion:** RPO 0 in-region and ≤ 1 min cross-region need **replication**; logical corruption and ransomware need **PITR + immutable, isolated backups**; restore speed for 4 TB must be measured, not assumed.

---

## 3. High-level design

```
                       Primary region ap-south-1                              DR region ap-south-2 (India, residency)
App pods ─▶ RDS Proxy ─▶ Aurora PostgreSQL cluster                          Aurora Global DB secondary cluster
                         writer (AZ-a) + readers (AZ-b, AZ-c)  ══ async ══▶  (1 reader instance, promotable)
                         storage: 6 copies across 3 AZs (quorum)            
                         PITR 35 days (continuous)
                              │ automated + manual snapshots
                              ▼
                 AWS Backup plan: daily + monthly ──copy──▶ Backup account (separate AWS account, different region)
                                                            Backup Vault with Vault Lock (compliance mode)
                                                            daily ×35, monthly ×84 (7 years), encrypted with backup-account KMS key
                 Monthly logical export of financial tables (Parquet) → S3 Object Lock (7 years)
Restore-test pipeline (weekly): restore latest snapshot in a sandbox account → run integrity checks → record duration
```

---

## 4. Deep dives

### 4.1 Disaster-by-disaster mapping

| Disaster | Mechanism | RPO | RTO |
|---|---|---|---|
| Instance failure | Aurora failover to reader in another AZ | 0 | ~30 s (+ app reconnect) |
| AZ loss | Same (storage quorum survives 1 AZ loss) | 0 | ~30–60 s |
| Region loss | Aurora Global DB unplanned failover (promote secondary) | ~1 s typical (lag) | ~5–15 min incl. app switch |
| Logical corruption (bad migration at 14:02) | PITR clone to 14:01 → compare/repair or switch | Seconds before the event | 1–4 h depending on repair scope |
| Accidental cluster deletion | Deletion protection; final snapshot; backup-account copies | ≤ 24 h (or PITR if cluster exists) | Hours |
| Ransomware / account compromise (attacker deletes cluster + snapshots in prod account) | Restore from **Vault-Locked copies in backup account** into a clean account | ≤ 24 h | ≤ 24 h |
| Corrupt backups | Weekly restore tests catch it early | — | — |

### 4.2 Failover mechanics and fencing
- In-region: managed by Aurora; **RDS Proxy** keeps client connections and reduces failover impact; apps use retry with backoff + idempotency keys for payment writes.
- Cross-region: **managed planned switchover** for drills (no data loss); **unplanned failover** (detach + promote) for real disasters. Before promoting, **fence** the old primary if reachable (block app writes in the old region: scale down writers, revoke SG access) to avoid split brain (topic 26).
- Apps in the DR region read the writer endpoint from config (Aurora Global "global writer endpoint" or a DNS name we control with short TTL).

### 4.3 Logical corruption playbook
```
1. Stop the bleeding: roll back the deploy / stop the job causing writes; maybe put affected features read-only
2. Identify the time window (deploy annotation, audit logs, pg logs)
3. Aurora "restore to point in time" as a NEW cluster at T-1min (or fast database clone for recent errors)
4. Compare affected tables (rows changed after T) between clone and prod
5. Repair: targeted SQL to restore rows (preferred: keeps newer valid writes), reviewed by two people
   or switch the app to the clone if corruption is broad and data after T can be replayed (from outbox/events)
6. Reconcile with payment provider reports (topic 22)
```

### 4.4 Backups the attacker can't delete
- **Separate backup account** in a separate OU, no human standing access, break-glass only.
- **AWS Backup Vault Lock in compliance mode**: even the root user can't delete recovery points before retention ends.
- Backups encrypted with a KMS key owned by the backup account (copy jobs re-encrypt).
- Alerts on any `DeleteRecoveryPoint`, vault policy change attempts, failed copy jobs.

### 4.5 Long-term retention (7 years)
Monthly snapshots in the backup vault (84 recovery points), plus monthly logical exports of financial tables to S3 Object Lock → engine-version-independent, queryable with Athena for audits (snapshots from 7 years ago may need an engine upgrade path to restore).

---

## 5. Scaling considerations
- As data grows, restore time grows → track **measured restore duration** in the weekly test; set an alert when it exceeds 50% of the RTO budget.
- Archive cold data (old transactions) to a separate history DB/warehouse to keep the hot DB smaller (faster restores, cheaper).
- Read scaling via readers; global secondary can serve read-only reporting in DR region.

---

## 6. Reliability and DR testing
| Test | Frequency | Pass criteria |
|---|---|---|
| Automated snapshot restore + checks (row counts, checksums on key tables, `pg_amcheck`) | Weekly | Restore < 2 h, checks pass |
| PITR clone to a random timestamp | Monthly | Data matches expected |
| In-region failover (reboot with failover) | Monthly (low traffic) | Error spike < 60 s |
| Global DB planned switchover to DR and back | Quarterly | RTO < 30 min, RPO 0 |
| Restore from backup account into clean account (ransomware drill) | Twice a year | < 24 h |

---

## 7. Security
- Encryption at rest (KMS CMK) and in transit (`rds.force_ssl`).
- IAM auth or Secrets Manager rotated credentials per service; DBA access via SSO + just-in-time + session logging.
- Deletion protection, SCP denying deletion of tagged critical DBs without a break-glass role.
- Database activity streams / pgaudit → SIEM for access audit.

---

## 8. Observability
- Replication: `AuroraGlobalDBReplicationLag` (alert > 5 s, page > 30 s since RPO is 1 min), `AuroraReplicaLag`.
- Backups: AWS Backup job failures, copy failures (page if two consecutive daily copies fail), age of newest recovery point in backup account.
- Restore tests: duration trend and pass/fail.
- DB health: connections, CPU, freeable memory, deadlocks, long transactions, storage growth.

---

## 9. Cost (relative, illustrative)
| Item | Added cost |
|---|---|
| Aurora Global DB secondary (1 instance + replicated storage + I/O) | ~+35–50% of the primary DB cost |
| Backup storage (35 daily + 84 monthly in another region/account) | Moderate; incremental snapshots |
| Cross-region transfer | Small (~hundreds $/month) |
| Restore-test sandbox (runs a few hours weekly) | Small |
For a tier-0 payments DB, this is cheap compared with one hour of downtime.

---

## 10. Trade-offs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Aurora Global DB | RPO ~1 s, managed failover | Cross-region read replica (RDS) / backups only | RTO 30 min + RPO 1 min requires continuous replication |
| DR region in India | Residency | Singapore | Compliance |
| Vault Lock compliance mode | Ransomware-proof | Governance mode | Can't be bypassed, but also can't delete mistakes early (accepted) |
| Repair over full restore for logical errors | Keeps valid newer writes | Restore whole DB | Less data loss; needs careful SQL |

---

## 11. Follow-up questions

1. **Q: Why aren't Multi-AZ and the Global DB enough?**
   A: They replicate everything, including mistakes and malicious changes, within seconds. Logical corruption, deletion and ransomware need point-in-time and immutable copies outside the blast radius of the production account.

2. **Q: How do you know your RTO is real?**
   A: We measure it: weekly automated restores record duration, quarterly switchovers record end-to-end time including app reconnection, and we alert when measured times approach the target. Untested RTOs are guesses.

3. **Q: During regional failover, how do you avoid double-processing payments?**
   A: Writes use idempotency keys stored in the DB, so retries after failover return the stored result. Outbox events are idempotent on the consumer side. Reconciliation with the payment provider's reports catches anything in the RPO window.

4. **Q: The DBA ran `DELETE FROM payments` without `WHERE` 10 minutes ago. Walk me through recovery.**
   A: Stop writes to that table (feature flags/read-only mode), note the exact time from logs, create a PITR clone at one second before, copy the deleted rows back from the clone (excluding rows legitimately changed since, by comparing with audit/outbox data), verify counts and sums, re-enable writes, and reconcile with the provider. Then postmortem: remove standing DBA write access, require peer-reviewed change tickets for manual SQL.
