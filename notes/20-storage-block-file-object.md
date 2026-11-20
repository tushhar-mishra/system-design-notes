# 20 · Storage: Block, File, Object

> **Level:** ★★ intermediate
>
> **You will learn:** the three storage types (**block, file, object**) and when to use each, **IOPS, throughput and latency** for disks, **durability vs availability** (what "11 nines" really means), **storage classes / tiers** and **lifecycle rules**, **versioning, replication and object lock**, **ephemeral vs persistent** storage, how Kubernetes **PV/PVC/StorageClass** map onto these, and the storage failures that page you (full disks, IOPS throttling, AZ-bound volumes).
>
> **You need:** topics 03, 08, 16. [Linux 094 · storage I/O path](../../linux-mastery-rhel/notes/094-storage-io-path.md) and [K8s 38 · PV/PVC](../../CKA-Certification-Course-2025/notes/38-persistent-storage-pv-pvc.md) go deeper.
>
> **Part 3 · Caching and data** · Previous: [19 · Consistency](19-consistency-cap-pacelc.md) · Next: [21 · Queues and streaming](21-queues-pubsub-streaming.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.**

```
Block  = a raw virtual disk attached to ONE machine. You put a filesystem on it.   (EBS, Persistent Disk, Managed Disk)
         → databases, boot disks. Fast, low latency. Usually tied to one AZ.
File   = a shared filesystem many machines mount over the network (NFS/SMB).      (EFS, Filestore, Azure Files)
         → shared content, legacy apps, home dirs. Easy, but slower and pricier.
Object = files ("objects") in buckets via HTTP API. No mount, no in-place edit.     (S3, GCS, Blob Storage)
         → backups, logs, images, artifacts, data lakes. Huge scale, cheap, very durable.
```

**Picture it.** Block = your own private hard drive. File = a shared office network drive. Object = a giant warehouse where every box has a label: you hand in a box, get it back by its label, but you can't open a box on the shelf and change one page; you replace the whole box.

**The one thing to know.** **Default to object storage** for anything that is "files" (uploads, logs, backups, artifacts). Use block for databases and things that need a real filesystem with low latency. Use shared file storage only when several machines truly need the same POSIX filesystem.

**Try it now:** `aws s3api get-bucket-lifecycle-configuration --bucket <bucket>`: does your log bucket expire old data?

**You've got it if you can:** pick the storage type for 5 workloads, explain durability vs availability, and design a lifecycle policy.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| | Block | File | Object |
|---|---|---|---|
| Access | Attached disk (/dev/nvme1n1) | Network mount (NFS, SMB) | HTTP API (PUT/GET/DELETE) |
| Sharing | One instance (some multi-attach exceptions) | Many clients | Unlimited clients |
| Modify part of a file | Yes | Yes | No (rewrite whole object; multipart for big uploads) |
| Latency | Sub-ms to ms | ms | ~10–100 ms first byte |
| Scale | Up to tens of TB per volume | PB, elastic | Practically unlimited |
| Cost (rough) | Medium | Highest | Lowest |
| AWS / GCP / Azure | EBS / Persistent Disk, Hyperdisk / Managed Disks | EFS, FSx / Filestore / Azure Files, NetApp Files | S3 / Cloud Storage / Blob Storage |
| Typical use | Databases, boot, Kafka, Prometheus TSDB | Shared CMS content, ML datasets, legacy apps | Backups, logs, artifacts, media, Terraform state, data lake |

**Instance store / local NVMe** = disks physically on the host: fastest, but **ephemeral** (data lost when the instance stops or the host fails). Great for caches, scratch, and replicated systems (Cassandra, Kafka with replication).

**Disk performance terms:**
- **IOPS** (Input/Output Operations Per Second): how many small reads/writes per second. Databases care.
- **Throughput:** MB/s. Big sequential scans, backups, Kafka care.
- **Latency:** time per operation.
- AWS gp3 volumes: baseline 3,000 IOPS and 125 MB/s, provisionable independently of size (up to 80,000 IOPS / 2,000 MB/s on current limits); io2 for very high/consistent IOPS. **Instances also have their own EBS bandwidth limits**: a big disk on a small instance is capped by the instance.

---

## Why it exists

Different data needs different access. A database needs low-latency random writes (block). Ten web servers sharing uploaded files need a shared view (file, or better, object). Petabytes of logs and backups need cheap, durable, infinite storage (object). Picking wrong is expensive or slow.

---

## Analogy

Private drive / shared drive / warehouse (Core). Extras:
- **Storage classes** = the warehouse has a front shelf (fast, pricier), a back room (cheaper, slower) and an off-site vault (cheapest, hours to retrieve).
- **Lifecycle rules** = "move boxes untouched for 30 days to the back room; after a year to the vault; after 7 years shred them".

---

## How it works inside

### Durability vs availability

- **Durability** = the probability your data is **not lost**. S3 Standard is designed for **99.999999999%** (11 nines) per object per year by storing copies across ≥3 AZs and repairing automatically. If you store 10 million objects, you might expect to lose one every ~10,000 years.
- **Availability** = the probability you can **access** it right now. S3 Standard: 99.99% designed availability (SLA 99.9%).
- A system can be durable but temporarily unavailable (a region issue) or available but not durable (a single disk).

**EBS** volumes are replicated **within one AZ** (designed for 99.8–99.9% annual durability for gp3; io2 higher) and **live in one AZ**: a pod using an EBS-backed PV can only run in that AZ. → snapshots (stored in S3, regional) for durability and moving across AZs.

### Object storage features you design with

| Feature | What it does | Use |
|---|---|---|
| **Storage classes** | Standard → Infrequent Access → Glacier Instant/Flexible/Deep Archive (AWS); Standard/Nearline/Coldline/Archive (GCP); Hot/Cool/Cold/Archive (Azure) | Cost by access frequency |
| **Lifecycle rules** | Transition or expire by age/prefix/tag | Logs, backups, artifacts |
| **Intelligent-Tiering** | Auto-moves objects between tiers by access | Unknown access patterns |
| **Versioning** | Keeps old versions on overwrite/delete | Protects against mistakes; Terraform state buckets |
| **Replication** (same-region / cross-region) | Async copy to another bucket | DR, compliance, other accounts |
| **Object Lock** (WORM: Write Once Read Many) | Prevents delete/overwrite until a retention date | **Ransomware-proof backups**, compliance |
| **Pre-signed URLs** | Time-limited direct upload/download | Clients upload straight to S3, bypassing your app |
| **Multipart upload** | Upload big files in parallel parts | Files > ~100 MB |
| **Event notifications** | Emit events on object create | Trigger processing (thumbnailing → queue) |
| **Encryption** | SSE-S3 (default), SSE-KMS (your key, audit) | Compliance (topic 40) |

**Cost traps:** minimum storage duration and retrieval fees in cold tiers (e.g. Glacier Deep Archive: 180-day minimum, hours to restore); per-request charges for millions of tiny objects; egress.

**Performance:** S3 scales per prefix (at least 3,500 writes and 5,500 reads per second per prefix); spread hot data across prefixes; use parallel ranged GETs for big objects.

### Direct-to-object-storage upload flow

```
1. Client → App: "I want to upload photo.jpg"
2. App → S3 SDK: create pre-signed PUT URL (valid 5 min, max size)
3. App → Client: URL
4. Client ──PUT bytes──▶ S3   (app servers never touch the bytes → no bandwidth/CPU on app tier)
5. S3 → event → queue → worker: scan, resize, record metadata in DB
```

### Kubernetes mapping

| K8s object | Maps to |
|---|---|
| `emptyDir` | Node disk / memory, deleted with the pod ([K8s 37](../../CKA-Certification-Course-2025/notes/37-ephemeral-volumes.md)) |
| PVC with EBS CSI StorageClass (`gp3`) | Block volume, **ReadWriteOnce**, AZ-bound |
| PVC with EFS CSI | File, **ReadWriteMany** |
| Object storage | Not a volume: apps use the S3 API (IRSA / Pod Identity for credentials) |

StorageClass `volumeBindingMode: WaitForFirstConsumer` creates the volume in the AZ where the pod is scheduled (avoids "volume in AZ-a, pod scheduled in AZ-b") ([K8s 39](../../CKA-Certification-Course-2025/notes/39-storageclasses-dynamic-provisioning.md)).

---

## Options and trade-offs

| Workload | Choose | Why / trade-off |
|---|---|---|
| Postgres on VM/K8s | Block (gp3/io2), or managed DB (better) | Low latency; AZ-bound → replicate at DB level |
| User uploads, media | Object + CDN | Cheap, durable, scalable; no shared mount |
| Shared config/content across many pods (legacy) | File (EFS) | Easy POSIX; latency and cost higher; watch throughput mode |
| Logs and metrics long-term | Object with lifecycle | Cheapest for big volumes (topics 49–50) |
| Backups | Object, versioned, **cross-region/cross-account copy, Object Lock** | Survives region loss and ransomware |
| CI caches, build artifacts | Object (S3) with expiry | Cheap; parallel access by runners |
| Kafka, Elasticsearch, Prometheus local TSDB | Block (or local NVMe + replication) | Throughput and latency |
| Scratch/temp | Instance store / emptyDir | Fast, free, ephemeral |

---

## Failure modes

| Failure | Symptom | Detect | Fix / prevent |
|---|---|---|---|
| **Disk full** | DB stops writing, apps crash, logs lost | Free space alerts at 80/90%, forecast "full in 4 h" alert | Autoscaling storage, log rotation ([Linux 038](../../linux-mastery-rhel/notes/038-logging-journald-rsyslog-logrotate.md)), retention |
| **IOPS/throughput throttling** | Latency spikes, high iowait, queue depth up | `VolumeQueueLength`, burst balance, `iostat` | Provision IOPS (gp3/io2), bigger instance (EBS bandwidth), spread I/O |
| **Pod can't start: volume in another AZ** | `volume node affinity conflict`, Pending | Pod events | `WaitForFirstConsumer`, per-AZ StatefulSets, replicate at app level |
| **EBS volume lost / AZ outage** | Data on that volume unavailable | AZ events | Snapshots, DB replication across AZs |
| **Accidental delete / overwrite in S3** | Missing data | Access logs, data checks | Versioning, MFA delete, Object Lock, replication to separate account |
| **Bucket made public** | Data leak | Config rules, Access Analyzer | Block Public Access at account level (topic 40) |
| **Glacier restore too slow for RTO** | Recovery takes hours/days | DR drill | Keep recent backups in a fast tier (topic 41) |
| **Millions of tiny objects** | Request cost, slow listing | Bill, LIST latency | Batch/compact small files |
| **EFS throughput exhaustion** (bursting mode) | Slow file ops after burst credits drain | `BurstCreditBalance` | Elastic/provisioned throughput |

**3 AM:** "Database writes failing." Check free storage first: a disk-full DB is a classic. Immediate: grow the volume (EBS can grow online; filesystem resize after), purge old logs/WAL if safe. Prevent: alerts on time-to-full, storage autoscaling.

---

## Where you'll use it

- **Terraform state** in a versioned, encrypted S3 bucket ([Terraform 35](../../Terraform/notes/35-remote-backends.md)).
- **Container registries** store layers in object storage (topic 47, [Docker 46a · registry ops](../../Docker/notes/46a-registry-operations.md)).
- **Jenkins/GHA artifacts and caches** in S3.
- **Backups** of etcd, databases, volumes (Velero for K8s) → S3 with Object Lock (topic 41).
- **Docker volumes on hosts** ([Docker 24](../../Docker/notes/24-volumes.md), [Docker 48 · disk problems](../../Docker/notes/48-storage-and-disk-problems.md)).

---

## Design exercise

**Prompt:** Design storage for an internal CI platform: (a) build caches (dependency caches, 2 TB, frequently read, can be rebuilt), (b) build artifacts (binaries, 500 GB/week, kept 90 days, release artifacts forever), (c) build logs (keep 30 days hot, 1 year for audit), (d) a shared Maven/npm proxy cache used by 200 runners.

<details>
<summary><b>Model answer</b></summary>

- **(a) Build caches:** S3 bucket, prefix per repo/branch, lifecycle **expire after 14 days** of no use (cheap to rebuild). Standard class (frequent access). Same region as runners, accessed via S3 gateway endpoint (no NAT cost, topic 08).
- **(b) Artifacts:** S3 with versioning; lifecycle: non-release prefixes expire at 90 days; `releases/` prefix → Standard-IA after 30 days, **never expire**, replicated to a second region; Object Lock (governance mode) on releases to prevent tampering (supply-chain integrity, topic 40).
- **(c) Logs:** S3; Standard for 30 days → Glacier Instant Retrieval (or Flexible Retrieval) until 365 days → expire. Index metadata (build ID → key) in the CI DB.
- **(d) Package proxy:** an artifact manager (Nexus/Artifactory, or AWS CodeArtifact) whose storage backend is S3 (object) and whose metadata DB is Postgres; the service runs with ≥2 replicas behind a load balancer. **Not** an EFS mount shared by 200 runners (slow metadata ops, throughput limits).
- **Cost check:** 500 GB/week × 13 weeks ≈ 6.5 TB of artifacts at any time plus releases; at roughly $0.023/GB-month for S3 Standard (illustrative), that's about $150/month. Cheap; the bigger risk is NAT/egress if endpoints are missing.
</details>

---

## Quick recap

- Block = one machine's disk (DBs); file = shared mount (only when needed); object = HTTP buckets (default for files, backups, logs, artifacts).
- Durability (not losing data) ≠ availability (reaching it now). S3: 11 nines durability across AZs; EBS: one AZ.
- Lifecycle rules + storage classes control cost; watch minimum durations and retrieval time vs RTO.
- Versioning, cross-account replication and Object Lock protect against mistakes and ransomware.
- In K8s: EBS PVs are AZ-bound (RWO), EFS is RWX, object storage via API with workload identity.

### Interview questions

1. **Q: Ten web servers need access to user-uploaded images. EFS or S3?**
   A: S3 (object storage) with a CDN in front. Clients upload directly with pre-signed URLs, so the app tier never handles bytes; S3 is cheaper, more durable, scales without limits, and the CDN serves reads. EFS works but costs more, adds latency and throughput limits, and couples servers to a shared mount.

2. **Q: What does 11 nines of durability mean, and does it mean your data is safe?**
   A: It's the designed probability of not losing an object to infrastructure failure in a year, achieved by redundant copies across AZs. It doesn't protect against you: a buggy job or attacker deleting/encrypting objects. For that you need versioning, Object Lock, and copies in a separate account/region.

3. **Q: A StatefulSet pod is Pending after a node failure. Why?**
   A: Its EBS-backed PV lives in one AZ; if no schedulable node exists in that AZ (or the volume is still attached to the dead node), the pod can't start. Fixes: keep node capacity in every AZ, use `WaitForFirstConsumer`, rely on application-level replication across AZs for stateful systems, and know the force-detach procedure.
