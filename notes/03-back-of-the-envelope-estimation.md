# 03 · Back-of-the-Envelope Estimation

> **Level:** ★ foundation
>
> **You will learn:** how to turn "10 million users" into **requests per second, storage, bandwidth and server counts** in two minutes with round numbers; the **powers of 2** and **unit tricks** that make it easy; the **latency numbers** every engineer should know; **peak vs average**; rough **capacity of common components**; a first taste of **availability nines**; and four fully worked examples (a photo app, a logging platform, a CI/CD fleet, a video service).
>
> **You need:** topics 01–02.
>
> **Part 0 · How to think** · Previous: [02 · The map](02-request-journey-map.md) · Next: [04 · Networking essentials](04-networking-essentials.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** You don't need exact numbers. You need the **right order of magnitude** (10? 1,000? 100,000?) to know which design is required. Round aggressively.

**The four magic facts:**

```
1 day ≈ 86,400 s ≈ 100,000 s (10^5)       → 1 million per day ≈ 12 per second
Peak ≈ 2–3× average (10× for spiky events)
1 KB = 10^3 bytes, 1 MB = 10^6, 1 GB = 10^9, 1 TB = 10^12, 1 PB = 10^15
Bandwidth: bytes/s × 8 = bits/s           → 125 MB/s = 1 Gbps
```

**The method:**

```
users/day × actions/user  = actions/day
actions/day ÷ 100,000     = average per second
× peak factor (2–3)       = peak per second       → how many servers / DB size
actions/day × size        = storage per day       → × 365 × years = total storage
reads/s × response size   = bandwidth             → do we need a CDN?
```

**Picture it.** Planning a wedding. You don't know exactly how many guests will eat two plates. You do know ~300 guests × ~1.5 plates ≈ 450 plates, so you book a caterer for 500, not 50 or 5,000.

**The one thing to know.** Estimation exists to answer **"which part is hard?"**. 200 writes/s → one database is fine. 200,000 writes/s → you need sharding or a log-based design. Say that conclusion out loud.

**Try it now:** 50 M users, each makes 10 requests a day. Average and peak QPS? *(Answer: 500 M/day ÷ 10^5 = 5,000/s average, ~15,000/s peak.)*

**You've got it if you can:** convert daily counts to per-second in your head, size storage for 5 years, convert MB/s to Gbps, and say what the result means for the design.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**Back-of-the-envelope estimation** = quick, rough calculation using round numbers and simple assumptions, done on a whiteboard in a couple of minutes. The name comes from scribbling on the back of an envelope.

Key words:

- **QPS** (Queries Per Second) or **RPS** (Requests Per Second): how many requests arrive each second. Same idea.
- **DAU** (Daily Active Users) / **MAU** (Monthly Active Users).
- **Throughput:** how much work gets done per second (requests/s, MB/s).
- **Peak factor:** how much busier the busiest moment is than the average.
- **Order of magnitude:** the power of 10 (is it ~10, ~1,000, ~1,000,000?).

---

## Why it exists

Designs depend on scale. The same feature is a different system at different sizes:

| Writes per second | What it needs |
|---|---|
| ~10 | Any database on a small VM |
| ~1,000 | A well-tuned managed relational DB |
| ~10,000 | A large DB, careful indexing, batching, maybe a queue in front |
| ~100,000+ | Sharding or a distributed database / log (Kafka, Cassandra, DynamoDB) |

Without numbers you either **under-build** (it falls over) or **over-build** (you burn money and add complexity). Interviewers also use it to check you're grounded: a candidate who proposes Kafka for 5 events per second has shown poor judgement.

---

## Analogy

**Planning a road trip.** 1,200 km, average 60 km/h with stops = 20 hours of driving → two days → one hotel night → fuel ~1,200/15 km per litre ≈ 80 litres. No GPS precision needed; you now know you need a hotel booking and a fuel budget. Estimation tells you **which bookings to make**.

---

## How it works inside

### Step 0: Units and powers you must know

| Power of 2 | Exact | Approx | Name |
|---|---|---|---|
| 2^10 | 1,024 | 10^3 thousand | KB (kibibyte KiB) |
| 2^20 | 1,048,576 | 10^6 million | MB (MiB) |
| 2^30 | ~1.07 × 10^9 | 10^9 billion | GB (GiB) |
| 2^40 | ~1.1 × 10^12 | 10^12 trillion | TB (TiB) |
| 2^50 | ~1.13 × 10^15 | 10^15 | PB (PiB) |

In estimation, treat 1 KB = 1,000 bytes. The 2–10% difference doesn't matter.

**Typical sizes:**

| Thing | Size |
|---|---|
| A character (ASCII) | 1 byte |
| A UUID / ID | 16 bytes (36 as text) |
| A short DB row (user, order) | 0.5–1 KB |
| A tweet-like post with metadata | ~1 KB |
| A log line | 200–500 bytes |
| A Prometheus sample on disk (compressed) | ~1–2 bytes |
| A JSON API response | 1–50 KB |
| A compressed photo | 200 KB – 2 MB |
| 1 minute of HD video | ~50–100 MB |
| A container image | 50 MB – 1 GB |

**Time:**
- 1 day = 86,400 s → use **10^5** (it's 16% off; fine).
- 1 month ≈ 2.6 M s (≈ 2.5 × 10^6). 1 year ≈ 31.5 M s (≈ 3 × 10^7).
- **Rule:** 1 M per day ≈ 12/s. 100 M per day ≈ 1,200/s. 1 B per day ≈ 12,000/s.

**Bandwidth:** network is measured in **bits** per second, storage in **bytes**. 1 byte = 8 bits.
- 1 Gbps (gigabit per second) = 125 MB/s. 10 Gbps = 1.25 GB/s.

### Step 1: Latency numbers every engineer should know

These are **orders of magnitude** (2020s hardware; exact values vary by hardware and cloud).

| Operation | Time | Feel |
|---|---|---|
| CPU L1 cache reference | ~1 ns | instant |
| Main memory (RAM) reference | ~100 ns | 100× L1 |
| Read 1 MB sequentially from RAM | ~10–50 µs | |
| Round trip inside one data centre / same AZ | ~0.1–0.5 ms | |
| Round trip between AZs in one region | ~0.5–2 ms | |
| NVMe SSD random read (4 KB) | ~20–100 µs | |
| Read 1 MB sequentially from SSD | ~0.3–1 ms | |
| HDD (spinning disk) seek | ~5–10 ms | 100× SSD |
| Read 1 MB sequentially from HDD | ~5–10 ms | |
| Round trip same continent (e.g. Mumbai ↔ Singapore) | ~30–70 ms | |
| Round trip across oceans (e.g. Mumbai ↔ US East) | ~180–250 ms | |
| TCP + TLS 1.3 connection setup | 2 round trips | 400 ms+ across oceans! |

> **µs** = microsecond = one millionth of a second. **ns** = nanosecond = one billionth. **ms** = millisecond = one thousandth.

**What to take away:**
1. Memory is ~1,000× faster than a network round trip in the same data centre. → caching works.
2. Cross-region and cross-ocean round trips are huge. → CDNs, regional deployments, and **never** make chatty synchronous calls across regions.
3. Disk seeks on HDD are slow. → SSDs for databases; HDD/object storage only for sequential/cold data.
4. Light in fibre travels ~200,000 km/s → ~5 ms per 1,000 km one way. Physics sets a floor no engineering can beat.

### Step 2: Rough capacity of common components

Very rough, for **simple operations on reasonable hardware**. Always say "depends on the workload; I'd load test" (topic 31).

| Component | Rough capacity per node |
|---|---|
| Stateless app server (simple API, 4 vCPU) | 500 – 5,000 req/s |
| Nginx / Envoy proxy | 10,000 – 50,000+ req/s |
| PostgreSQL / MySQL (simple indexed queries, big instance) | 5,000 – 50,000 queries/s reads; 1,000 – 10,000 writes/s |
| Redis (single thread for commands) | ~100,000 ops/s |
| Kafka broker | ~50 – 100+ MB/s write per broker (depends on disks/network) |
| A single SQS standard queue | Practically unlimited (AWS scales it) |
| Elasticsearch/OpenSearch data node | ~5 – 20 MB/s indexing |

### Step 3: The estimation recipe

1. **Write assumptions** (DAU, actions per user, sizes, read:write ratio). Say them out loud.
2. **Traffic:** daily → average/s → peak/s, separately for reads and writes.
3. **Storage:** per item × items per day × retention. Add replication/backup overhead if it's yours to manage.
4. **Bandwidth:** in (writes × size) and out (reads × size).
5. **Memory for cache:** often "cache the hot 20%" of the daily read set (the 80/20 rule: a small part of the data gets most reads).
6. **Servers:** peak QPS ÷ per-server capacity × safety margin (×1.5–2, and N+1 for failure).
7. **Conclusion:** "So the hard part is X."

### Worked example 1: a photo-sharing app

Assumptions: 10 M DAU. Each user views 20 photos and uploads 0.2 photos per day. Photo = 500 KB. Metadata row = 1 KB. Keep photos forever. 5-year view.

```
Reads:   10 M × 20   = 200 M/day ÷ 10^5 = 2,000/s avg  → peak ×3 ≈ 6,000/s
Writes:  10 M × 0.2  =   2 M/day ÷ 10^5 =    20/s avg  → peak ×3 ≈    60/s
Ratio:   100 : 1 read-heavy → cache + CDN, writes are easy

Storage (photos):  2 M × 500 KB = 1 TB/day → ×365 ≈ 365 TB/year → ~1.8 PB in 5 years
Storage (metadata): 2 M × 1 KB = 2 GB/day → ~3.6 TB in 5 years → fits one DB (with care)

Egress:  2,000 views/s × 500 KB = 1 GB/s ≈ 8 Gbps avg, ~24 Gbps peak
```

**Conclusions:**
- Photos go to **object storage** (S3-like), not a database or local disks: 1.8 PB.
- 8–24 Gbps of egress → a **CDN** is mandatory (cost and latency). With a 95% CDN hit ratio, origin sees only ~1.2 Gbps peak.
- Metadata at 60 writes/s and 6,000 reads/s → one relational primary + read replicas + cache is fine. **No sharding needed yet.**
- Cost driver: storage and egress, not compute. (Topic 43: lifecycle old photos to colder storage tiers.)

### Worked example 2: a central logging platform

Assumptions: 2,000 servers, each produces 5 GB/day of logs. Average log line 250 bytes. Keep 14 days searchable, 1 year archived.

```
Volume:  2,000 × 5 GB = 10 TB/day
Rate:    10 TB / 10^5 s = 100 MB/s avg  → peak ×3 ≈ 300 MB/s  (≈ 2.4 Gbps)
Lines:   100 MB/s ÷ 250 B ≈ 400,000 lines/s avg, ~1.2 M/s peak

Hot (searchable) 14 days:  10 TB × 14 = 140 TB raw
   Indexed search engines often use ~1–1.5× raw size incl. replica  → ~150–300 TB of SSD
Archive 1 year in object storage, compressed ~10×:  10 TB × 365 / 10 ≈ 365 TB
```

**Conclusions:**
- 300 MB/s peak into an index needs a **buffer** (Kafka or similar) so index slowdowns don't drop logs.
- At ~10 MB/s per indexing node, you'd need ~30 nodes at peak. Expensive. → consider a cheaper index-light design (e.g. Loki-style label indexing) or **reduce volume** (drop debug logs, sample).
- Storage of 140 TB hot is the biggest cost → shorter hot retention or tiering. Full design: topic 49.

### Worked example 3: CI/CD runner fleet for 500 engineers

Assumptions: 500 engineers, each triggers ~6 pipeline runs per working day. Each run = 10 minutes of runner time. Work happens in an 8-hour window. Busiest hour ≈ 2× average hour.

```
Runs/day:      500 × 6 = 3,000
Per hour:      3,000 / 8 = 375/h avg → peak ≈ 750/h = 12.5/min
Concurrency:   arrivals per minute × minutes per run
               = 12.5 × 10 ≈ 125 runners busy at peak
```

The last line uses **Little's Law**: *items in the system = arrival rate × time each spends inside*. (Full story in topic 31.) If 12.5 builds start per minute and each takes 10 minutes, about 125 are running at any moment.

**Conclusions:**
- Need ~125 concurrent runners at peak (plus ~20% margin), but maybe ~10 at night → **autoscaled, ephemeral runners** (topic 47), not a fixed fleet.
- Runner-hours/day: 3,000 × 10 min = 500 h. On 4 vCPU instances at ~$0.20/h, compute is ~$100/day. Caching to cut runs from 10 → 6 minutes saves 40%.

### Worked example 4: video upload service (bandwidth-heavy)

Assumptions: 1 M uploads/day, 50 MB each, viewed 100× each on average, 3 renditions (sizes) stored at ~1.5× original total.

```
Upload in:   1 M × 50 MB = 50 TB/day ÷ 10^5 = 500 MB/s ≈ 4 Gbps avg in
Storage:     50 TB × 1.5 = 75 TB/day → ~27 PB/year
Views out:   100 M views × ~50 MB = 5 PB/day ≈ 50 GB/s ≈ 400 Gbps avg
```

**Conclusions:** egress of 400 Gbps is only possible through a big **CDN**; storage growth of 27 PB/year forces lifecycle policies (cold tier for old videos) and deduplication. Compute for transcoding is a **queue + worker** problem (topic 21).

### First taste: availability numbers

Availability is usually written in **nines** (full math in topic 23):

| Availability | Downtime per year | Per 30-day month |
|---|---|---|
| 99% ("two nines") | ~3.65 days | ~7.2 hours |
| 99.9% | ~8.76 hours | ~43.8 minutes |
| 99.95% | ~4.38 hours | ~21.9 minutes |
| 99.99% | ~52.6 minutes | ~4.4 minutes |
| 99.999% | ~5.3 minutes | ~26 seconds |

Quick method: 1 year ≈ 525,600 minutes. Downtime = (1 − availability) × 525,600. For 99.9%: 0.001 × 525,600 ≈ 526 min ≈ 8.76 h.

---

## Options and trade-offs: how precise should you be?

| Approach | When | Risk |
|---|---|---|
| Single round number per step (this topic) | Interviews, early design | Off by 2–3× (acceptable) |
| Spreadsheet model with growth rates | Capacity planning at work (topic 31) | Assumptions still guesses |
| Load test real system | Before launch, before big events | Takes time; test env ≠ prod |
| Production metrics + forecasting | Running systems | Needs history; surprises still happen |

**Average vs peak:** design capacity for **peak**, with **headroom** (spare capacity, usually 30–50%) and survive the loss of one failure domain (an AZ). Example: peak needs 12 servers; spread over 3 AZs and survive one AZ loss → 12 ÷ 2 AZs = 6 per AZ → **18 servers** total.

---

## Failure modes

| Mistake | Consequence | Prevention |
|---|---|---|
| Designing for average, not peak | Falls over every evening / every sale | Multiply by peak factor; ask about events (sales, launches) |
| Mixing bits and bytes | 8× wrong bandwidth | Write units on every number |
| Forgetting replication/backup overhead (self-managed) | Disks fill 3× faster | Raw × replicas × (1 + backup) |
| Forgetting index/metadata overhead | Search clusters 2× bigger than planned | Add 50–100% for indexes |
| Forgetting AZ-failure headroom | An AZ loss overloads survivors → total outage | N+1 per failure domain |
| Fake precision ("12,347.6 QPS") | Wastes time, looks naïve | Round: "~12 k QPS" |
| Never stating the conclusion | Numbers without meaning | End with "so the hard part is…" |

**3 AM:** the real-world version of bad estimation is **a capacity incident**: a marketing push 10× the normal traffic, disks filling, a queue that never drains. The fix is the same math done in advance: forecast, set alerts at 70–80% of capacity, and load test (topic 31).

---

## Where you'll use it

- **Every design interview** (Phase 2 in topic 01).
- **Capacity planning:** "When will this database run out of disk?" (topic 31).
- **Cost estimates:** "What will 14 days of log retention cost?" (topic 43).
- **Sanity checks during incidents:** "Is 40k QPS normal for 3 AM?"
- **Choosing instance sizes** in Terraform modules and K8s resource requests ([K8s 41 · requests and limits](../../CKA-Certification-Course-2025/notes/41-requests-limits-qos-quotas.md)).

---

## Design exercise

**Prompt:** A metrics platform scrapes 5,000 Kubernetes pods every 15 seconds. Each pod exposes 1,000 time series (a time series = one metric with one set of labels, e.g. `http_requests_total{path="/api",code="200"}`). Each sample is stored compressed in ~1.5 bytes. Estimate: samples per second, storage for 30 days, and say whether one Prometheus server (rule of thumb: comfortable up to a few million active series and ~1 M samples/s) is enough.

<details>
<summary><b>Model answer</b></summary>

```
Active series:   5,000 pods × 1,000 = 5 M series
Samples/s:       5 M / 15 s ≈ 333,000 samples/s
Per day:         333,000 × 86,400 ≈ 2.9 × 10^10 samples
Storage/day:     2.9 × 10^10 × 1.5 B ≈ 43 GB/day
30 days:         ~1.3 TB (+ index/WAL overhead, say ~1.5–2 TB)
```

Ingest of ~333 k samples/s is within one large Prometheus, but **5 M active series** is at the edge: memory use grows with series count (rough rule: a few KB of RAM per active series → ~15–30 GB RAM+). And one server is a single point of failure.

**Design conclusion:** run **two Prometheus replicas** (HA pair) or shard by namespace/cluster, and send data via remote write to a horizontally scalable long-term store (Thanos/Mimir/managed service) for 30-day+ retention. Also attack **cardinality**: 1,000 series per pod is high; dropping unused metrics often cuts it in half. (Full design: topic 50.)
</details>

---

## Quick recap

- Round hard: 1 day ≈ 10^5 s; 1 M/day ≈ 12/s; peak ≈ 2–3× average; 1 Gbps = 125 MB/s.
- Recipe: assumptions → traffic (avg, peak; reads, writes) → storage → bandwidth → cache memory → servers → conclusion.
- Know the latency ladder: RAM ≪ same-AZ network ≪ SSD ≪ HDD ≪ cross-ocean.
- Plan for peak + headroom + loss of one AZ.
- Always end with "so the hard part is…".

### Interview questions

1. **Q: 100 M requests a day, 80% reads. How many app servers?**
   A: 100 M ÷ 10^5 ≈ 1,000/s average, ~3,000/s peak. If one server handles ~500/s for this workload (I'd confirm with a load test), that's 6 at peak; add headroom and AZ failure: spread over 3 AZs and survive losing one → 6 ÷ 2 = 3 per AZ → 9 servers, autoscaled with a minimum of maybe 6.

2. **Q: How do you estimate cache size?**
   A: Take the daily read set, assume a small hot fraction gets most traffic (often 20%), and multiply by item size. 10 M distinct items read per day × 20% × 2 KB ≈ 4 GB → a small Redis cluster. Then verify with the real hit ratio after launch.

3. **Q: Why does cross-region synchronous replication hurt so much?**
   A: Every write must wait for a round trip to the other region, e.g. 70–200 ms, compared with ~1 ms within a region. That caps write latency and throughput. So most systems replicate across regions asynchronously and accept a small RPO, keeping synchronous replication within a region across AZs.
