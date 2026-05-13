# System Design Quick-Read: Object Storage

---

## 1. The 1-Minute Pitch
* **What it is:** A specialized storage architecture designed to store and manage massive volumes of unstructured data (Blobs) in a flat namespace.
* **Mental Model:** Think of it as a massive, distributed Key-Value store where the Key is the file path and the Value is an immutable "bag of bytes."
* **System Placement:** Sits alongside your primary database (RDBMS/NoSQL) to handle heavy, static assets while the DB manages structured metadata.
* **When to think of it:** * **Large Files:** Storing photos, videos, audio, or logs.
  * **Database "Choke":** When your relational database is slowing down due to large binary objects (BLOBs) bloating backups and memory.

## 2. Core Fundamentals Cheat Sheet
* **Data Model:** Flat Namespace. Objects consist of the data itself, customizable metadata, and a unique identifier.
* **Consistency:** Modern systems (like S3) offer strong read-after-write consistency for new objects; deletes and updates can be eventually consistent.
* **Durability:** Designed for "11 Nines" (99.999999999%) through redundancy across multiple Availability Zones (AZs).
* **Scaling Model:** Virtually unlimited horizontal scaling. You pay for what you use, and the system handles petabyte-scale growth automatically.
* **Failure Model:** Self-healing via Erasure Coding or Replication. If a node or rack fails, the metadata service routes requests to healthy replicas.
* **Latency Profile:** High time-to-first-byte (TTFB) compared to local disk, but optimized for high-throughput streaming of large files.

## 3. Architecture & Mechanics



**3.1 The Write Path**
* **Immutable Writes:** You cannot modify a file in place. Updates involve overwriting the object or creating a new version, which eliminates complex locking mechanisms.
* **Multipart Upload:** For large files, the client splits the data into chunks, uploads them in parallel, and the storage service stitches them together upon completion.

**3.2 The Read Path**
* **Metadata Lookup:** The client queries a Metadata Service to find the object's ID and location.
* **Direct Streaming:** To save server bandwidth, the client often receives a **Pre-signed URL** to download the file directly from the storage node.

**3.3 Scaling & High Availability**
* **Routing:** Uses a flat structure and string-based keys to locate data directly, avoiding the $O(N)$ directory-tree traversal overhead of traditional file systems.
* **Replication:** Data is mirrored across multiple data centers and regions to protect against localized disasters.

**3.4 Operational Knobs**
* **Lifecycle Policies:** Automates moving "cold" data to cheaper tiers (like Archive/Glacier) or deleting logs after a retention period.
* **Versioning:** Keeps a history of object changes to protect against accidental deletes or overwrites.

## 4. Interview Use Cases
* **Common Patterns:**
  * **Metadata-Blob Separation:** Keep the post text in PostgreSQL and the post image in S3.
  * **Data Lakes:** Central repository for raw data used in Big Data analytics and Machine Learning.
  * **Static Asset Hosting:** Serving CSS, JS, and images via a CDN fronting the Object Store.
* **When to CHOOSE it:** * Need high durability and nearly infinite scale.
  * Storing data that is read often but rarely (or never) modified.
* **When to AVOID it:** * Low-latency, high-frequency random access (use Redis/Memcached).
  * Frequently updating small portions of a large file (use Block Storage/EBS).

## 5. Trade-offs, Pitfalls, & Alternatives
* **Common Gotchas (The "Senior" Signals):**
  * **The "Server Proxy" Bottleneck:** Never upload/download large files *through* your app server. Use Pre-signed URLs to offload bandwidth to the storage provider.
  * **RDBMS Bloat:** Storing 5MB images in a DB row creates massive memory pressure and turns a 5-minute DB restoration into a 5-hour nightmare.
* **Comparisons:**
  * **vs. File Storage (NFS):** File storage uses a hierarchy (folders) and is easier for shared mounting; Object storage is flat and scales orders of magnitude better.
  * **vs. Block Storage (EBS):** Block storage is "raw" disk for an OS/DB; Object storage is a high-level API-based service for bulk data.

## 6. The "Drop-In" Interview Script
> **Proposing it:** "I’ll introduce Object Storage here to handle user-uploaded videos because it decouples heavy binary data from our relational schema, ensuring high durability without affecting DB performance."
> **Justifying a feature:** "To handle the scale, I'll use **Multipart Uploads** for reliability and **Pre-signed URLs** so our app servers aren't throttled by file-transfer bandwidth."
> **Owning the trade-off:** "We’ll accept the higher latency of an API-based store because the data is largely static and the cost-to-scale ratio is unbeatable for this volume."

---

## 7. One-Minute Recap
* **Use when:** You need **infinite scale**, **extreme durability**, and **low cost** for unstructured files.
* **Do NOT use when:** You need **low-latency random access** or **in-place file edits**.
* **Key strength:** Breaking down data silos with a globally accessible, flat namespace.
* **Key weakness:** Not designed for high-frequency transactional updates.