# Back-of-the-Envelope (BOTE) Estimation

> Quick-reference guide for modern system design interviews (2025–2026).

***

## Overview

| Section | Focus                                      |
|--------:|--------------------------------------------|
| 1       | [What interviewers actually test](#1-the-interviewers-true-goal) |
| 2       | [Golden rules for fast estimations](#2-the-golden-rules-of-estimation) |
| 3       | [Storage, size, and time fundamentals](#3-core-fundamentals-to-memorize) |
| 4       | [Traffic and bandwidth cheat codes](#4-the-magic-cheat-codes) |
| 5       | [Four-step estimation framework](#5-the-four-pillars-of-estimation) |
| 6       | [Modern latency and availability numbers](#6-modern-system-design-trends-20252026) |
| 7       | [End-to-end Twitter/X clone walkthrough](#7-example-walkthrough-twitterx-clone) |

Use this as a **visual, skimmable cheat sheet** rather than a long-form article.

***

## 1. The Interviewer's True Goal

They are not testing your ability to do complex mental math. They are testing your ability to:

- [x] **Reason under uncertainty:** Translate abstract requirements into concrete architecture.
- [x] **Identify bottlenecks:** Decide whether you need to partition the database, introduce a cache, or change the data model.
- [x] **Communicate clearly:** State assumptions, justify them, and pivot when given new constraints.

> **Framing tip:** Say out loud: "I will make some reasonable assumptions, do quick back-of-the-envelope math, and refine if constraints change."

***

## 2. The Golden Rules of Estimation

### 2.1 Visual Rules Summary

| Rule                    | One-line memory hook                              |
|-------------------------|---------------------------------------------------|
| Round aggressively      | Replace weird numbers with clean powers of ten    |
| 1000x noise rule        | Ignore the tiny term when one value dominates     |
| State assumptions       | Say your DAU, ratios, and sizes out loud          |
| Always label units      | Every number should have a unit next to it        |

### 2.2 Round aggressively

Precision is the enemy of speed.  
Never calculate `99,987 / 9.1`. Round it to **100,000 / 10 = 10,000**.

> **Rule of thumb:** Prefer answers that are clearly *correct in magnitude* over answers that are *exact but slow*.

### 2.3 The 1000x noise rule

If you are adding two numbers that differ by 1000x or more, ignore the smaller one.  
Example: **1 PB + 1 TB ≈ 1 PB**.

This keeps math fast without losing directional accuracy.

### 2.4 State assumptions out loud

Always write down and vocalize your assumptions. For example:

```text
DAU = 10M
Requests per user per day = 5
Read:Write = 90:10
```

Make it a collaborative discussion with the interviewer.

### 2.5 Always label your units

Always write `5 MB`, not just `5`.  
This prevents catastrophic magnitude errors later in the calculation.

> **Anti-bug check:** When you see a bare number like `1000`, immediately ask yourself: "1000 **what**? Bytes? KB? QPS?".

***

## 3. Core Fundamentals to Memorize

### 3.1 Storage Cheat Sheet (BKMGTP)

Memorize this mnemonic. Each step is a multiple of 1,000 (use 1,000 instead of 1,024 for mental math).

```text
B  K  M  G  T  P
Byte, Kilo, Mega, Giga, Tera, Peta
```

| Power | Prefix   | Abbr | Approximate value           |
|------:|----------|:----:|-----------------------------|
| 10^0  | Byte     | B    | 1 byte (e.g., 1 ASCII char) |
| 10^3  | Kilobyte | KB   | 1 thousand bytes            |
| 10^6  | Megabyte | MB   | 1 million bytes             |
| 10^9  | Gigabyte | GB   | 1 billion bytes             |
| 10^12 | Terabyte | TB   | 1 trillion bytes            |
| 10^15 | Petabyte | PB   | 1 quadrillion bytes         |

### 3.2 Standard data sizes

| Item                              | Size (rough)           |
|-----------------------------------|------------------------|
| Char / Boolean                    | 1–2 bytes              |
| Integer / Unix timestamp          | 4 bytes                |
| Long / Double (64-bit ID)         | 8 bytes                |
| Short text post                   | ~200 bytes             |
| Typical user image                | ~300 KB                |
| High-resolution image             | 2–3 MB                 |
| Standard video per minute         | ~100 MB                |

> **Memory hook:**  `Text ≪ Image ≪ Video` in raw size.

### 3.3 Time Cheat Sheet

| Concept              | Exact value                    | Mental approximation      |
|----------------------|--------------------------------|---------------------------|
| Seconds in a day     | 24 × 60 × 60 = 86,400         | ≈ 100,000 seconds         |
| Days in a year       | 365                           | ≈ 400 days (rough bound)  |

***

## 4. The Magic "Cheat Codes"

Use these shortcuts to bypass tedious math during interviews.

### 4.1 Quick traffic and bandwidth table

| Scenario                     | Heuristic                           | Mental model                          |
|------------------------------|-------------------------------------|----------------------------------------|
| 1M requests per day          | ≈ 12 QPS                            | 1M ÷ 100k ≈ 10; adjust slightly        |
| 100M requests per day        | ≈ 1,200 QPS                         | 100 × (1M/day case)                    |
| 100 GB traffic per day       | ≈ 1 MB/s                            | 100 GB ÷ 100k s ≈ 1 MB/s               |

### 4.2 Shortcut 1: Traffic

**Rule of thumb:**

- **1 million requests/day ≈ 12 QPS (requests per second)**

Math: **1,000,000 ÷ 86,400 ≈ 11.5 ≈ 12**

Extension:

- **100 million requests/day ≈ 1,200 QPS**

### 4.3 Shortcut 2: Bandwidth

**Rule of thumb:**

- **100 GB of traffic/day ≈ 1 MB/s bandwidth**

Math: **100 GB ÷ 86,400 s ≈ 1.15 MB/s ≈ 1 MB/s**

> **Interview move:** Do the heuristic first, then say: "If we need more precision we can refine, but this is good enough for capacity planning."

***

## 5. The Four Pillars of Estimation

Break every estimation down into these four sequential steps.

### 5.1 Visual pipeline

```text
Users  →  Traffic (QPS)  →  Storage  →  Bandwidth  →  Cache
           (Load)            (Data)      (Network)    (Hot set)
```

### 5.2 Pillar I – Load / Traffic (QPS)

1. **Find Daily Active Users (DAU)**  
   Example: `DAU = 100M`.
2. **Determine read/write ratio**
    - Read-heavy example (YouTube-style): `100:1` (reads:writes)
    - Write-heavy example (web crawler): `1:1`
3. **Average QPS**  
   Approximate: **Average QPS ≈ (DAU × actions per user per day) ÷ 100,000**
4. **Peak QPS**  
   Approximate: **Peak QPS ≈ Average QPS × 2**

> **Sanity check:** If your QPS is less than 10 for a "planet-scale" product, your assumptions are off.

### 5.3 Pillar II – Storage

1. **Daily storage**  
   **Daily storage = daily writes × size of average request**
2. **Yearly storage**  
   **Yearly storage = daily storage × 365**
3. **Total storage (e.g., 5-year plan)**  
   **Total storage = yearly storage × 5**
4. **Replication**  
   Account for replication, e.g.: **Total physical storage ≈ logical storage × 3**

> **Design angle:** Once storage is in TB/PB, you can talk about sharding strategy, cold storage, and archival policies.

### 5.4 Pillar III – Bandwidth

- **Ingress (in):**  
  **Ingress = write QPS × size of average write**

- **Egress (out):**  
  **Egress = read QPS × size of average read**

> **Interpretation:** If egress is huge but storage is modest, you probably need a strong cache and CDN.

### 5.5 Pillar IV – Cache

- **80/20 rule:** Roughly 20% of data generates 80% of traffic.
- **Crucial point:** Calculate 20% of your **daily read volume**, not total storage.
- **Cache size formula:**  
  **Cache size ≈ daily read requests × average object size × 0.20**

```text
If total reads are huge but cacheable, RAM is often cheaper than overprovisioning origin databases.
```

***

## 6. Modern System Design Trends (2025/2026)

### 6.1 Latency ladder

Hardware has evolved. Use modern numbers to show you are up to date.

| Layer / Operation                            | Approximate latency |
|----------------------------------------------|---------------------|
| CPU L1/L2 cache access                       | ~0.5 ns – 7 ns      |
| Mutex lock/unlock                            | ~25 ns              |
| Main memory (RAM) reference                  | ~100 ns             |
| Modern NVMe SSD fsync                        | ~0.5 ms             |
| Legacy HDD disk seek                         | ~10 ms              |
| Same-datacenter network round trip           | ~0.5 ms             |
| Cross-continent network round trip (CA ↔ NL) | ~150 ms             |

> **Story pattern:** Map your critical path: "Request hits cache → DB → secondary index → object storage" and attach approximate latencies.

### 6.2 High availability and cloud SLAs

Uptime is typically measured in "nines." Be ready to discuss the trade-off between **cost** and **downtime**.

| Availability % | Uptime level | Downtime per year | Downtime per day |
|---------------:|-------------|-------------------|------------------|
| 99%            | Two 9s      | 3.65 days         | 14.40 minutes    |
| 99.9%          | Three 9s    | 8.77 hours        | 1.44 minutes     |
| 99.99%         | Four 9s     | 52.60 minutes     | 8.64 seconds     |
| 99.999%        | Five 9s     | 5.26 minutes      | 864.00 ms        |

> **Design angle:** Ask which components really need four or five 9s, and which can tolerate lower SLAs.

***

## 7. Example Walkthrough: Twitter/X Clone

Putting it all together on the whiteboard.

### 7.1 High-level snapshot

| Metric                       | Order of magnitude      |
|------------------------------|-------------------------|
| DAU                          | 150M users              |
| Write QPS                    | ~3,000 QPS              |
| Peak write QPS               | ~6,000 QPS              |
| Read QPS                     | ~300,000 QPS            |
| Daily storage (logical)      | ~30 TB/day              |
| 5-year storage (logical)     | ~55 PB                  |
| Ingress bandwidth            | ~300 MB/s               |
| Egress bandwidth             | ~30 GB/s                |
| Cache (hot set)              | ~600 TB RAM             |

### 7.2 Step 1: Assumptions

- **Users:** `300M MAU` → `150M DAU`
- **Activity:** `2 tweets/user/day`
- **Read/write ratio:** `100:1` (read-heavy)
- **Payload:**
    - 10% of tweets have media (~1 MB)
    - Remaining 90% are text-only (~200 bytes)

### 7.3 Step 2: Load (QPS)

- **Write QPS**  
  **Write QPS = (150M DAU × 2 tweets) ÷ 100,000 s ≈ 3,000 QPS**

- **Peak write QPS**  
  **Peak write QPS ≈ 3,000 × 2 = 6,000 QPS**

- **Read QPS**  
  **Read QPS = 3,000 write QPS × 100 = 300,000 QPS**

### 7.4 Step 3: Storage (5-year plan)

- **Text per day**  
  **Text/day = 150M × 2 × 200 B = 60 GB/day**

- **Media per day**  
  **Media/day = 150M × 2 × 10% × 1 MB = 30,000,000 MB ≈ 30 TB/day**

- **Total daily storage**  
  **Total/day ≈ 30 TB/day**

- **5-year storage (logical)**  
  **5-year storage ≈ 30 TB × 365 × 5 ≈ 55 PB**

(You can add replication separately if you want physical storage.)

### 7.5 Step 4: Bandwidth

- **Ingress (writes)**  
  Approximate average write size:

    - 90% text-only: 200 B
    - 10% media: 1 MB

  So: **Ingress ≈ 3,000 QPS × (0.9 × 200 B + 0.1 × 1 MB) ≈ 300 MB/s**

- **Egress (reads)**  
  Assume average read payload of ~100 KB:  
  **Egress ≈ 300,000 QPS × 100 KB ≈ 30 GB/s**

### 7.6 Step 5: Cache

- **Daily reads**  
  **Daily reads ≈ 30 TB (daily writes) × 100 (read:write ratio) = 3 PB/day**

- **Cache capacity (80/20 rule)**  
  **Cache needed ≈ 3 PB × 0.20 = 600 TB of hot data in RAM**

```text
Story you can tell:
"Given these numbers, I would shard tweets by user ID, store media in object storage,
put a large Redis or Memcached layer in front of the primary DB, and rely heavily on a CDN for fan-out reads."
```