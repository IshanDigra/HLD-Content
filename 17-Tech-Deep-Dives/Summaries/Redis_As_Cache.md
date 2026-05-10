# System Design Quick-Read: Redis (Cache)

---

## 1. The 1-Minute Pitch
* **What it is:** A high-performance, in-memory key-value data store primarily used to reduce database load and lower read latency by serving frequently accessed data from RAM.
* **Mental Model:** Think of it as a globally accessible, extremely fast, and distributed Hash Map with built-in expiration logic.
* **System Placement:** Sits between the application servers and the primary database (e.g., PostgreSQL, DynamoDB) to intercept read requests.
* **When to think of it:** * **Heavy read traffic:** When your primary DB is throttled or expensive to scale for high-frequency queries.
  * **Need to offload primary DB:** When expensive computations or complex lookups (like product details or session data) can be pre-computed and stored.

## 2. Core Fundamentals Cheat Sheet
* **Data Model:** Key-Value; supports complex types like Strings (JSON blobs), Hashes (field-value pairs), Lists, Sets, and Sorted Sets.
* **Consistency:** **Eventual Consistency** (in a cache-aside pattern) relative to the DB; on a single node, it provides strong consistency for operations.
* **Durability:** Primarily **In-memory**; optionally persistent via RDB (snapshots) or AOF (append-only log), though often disabled or tuned down when used strictly as a cache.
* **Scaling Model:** **Redis Cluster (Sharding)**. Data is partitioned across multiple nodes using hash slots to scale memory and throughput.
* **Failure Model:** **Leader-Follower replication**. Automatic failover via Redis Sentinel or Cluster mode; handles network partitions by promoting followers to leaders.
* **Latency Profile:** **Sub-millisecond**. RAM-based access ensures extremely low response times regardless of dataset size (assuming it fits in memory).

## 3. Architecture & Mechanics
**3.1 The Write Path (Cache-Aside Pattern)**
* Application receives a write request.
* Update the primary Database first.
* Invalidate (delete) or Update the corresponding key in Redis.
* **Durability:** Writes to RAM are immediate; async replication to followers occurs in the background.

**3.2 The Read Path**
* Application checks Redis first:
  * **Cache Hit:** Return data immediately (latency ~0.1-1ms).
  * **Cache Miss:** Query primary DB, populate Redis with the result, then return to user.
* **Efficiency:** Uses an asynchronous, non-blocking I/O model (event loop) to handle thousands of concurrent connections.

**3.3 Scaling & High Availability**
* **Routing/Sharding:** Redis Cluster uses 16,384 hash slots. Keys are mapped to slots via `CRC16(key) mod 16384`.
* **Replication:** Uses asynchronous leader-follower replication to provide high availability for reads and failover support.

**3.4 Operational Knobs**
* **Eviction/Retention:** Uses **TTL (Time-To-Live)** to expire data. When memory is full, it uses policies like **allkeys-lru** (Least Recently Used) or **volatile-lru** to reclaim space.
* **Consistency Tuning:** TTL ensures that even if invalidation logic fails, the cache eventually converges with the DB.

## 4. Interview Use Cases
* **Common Patterns:** Session management, Product Catalog caching (JSON blobs), Rate limiting (counters), and Leaderboards (Sorted Sets).
* **When to CHOOSE it:** * **High read QPS:** You need to scale to hundreds of thousands of reads per second.
  * **Strict Latency Requirements:** Your p99 latency budget is extremely tight (e.g., real-time bidding).
* **When to AVOID it:** * **Strict ACID transaction requirements:** If you need cross-shard atomicity or complex relational joins.
  * **Datasets larger than RAM:** If the data doesn't fit in memory and doesn't benefit from a "working set" pattern, a disk-based DB is better.

## 5. Trade-offs, Pitfalls, & Alternatives
* **Common Gotchas (The "Senior" Signals):**
  * **Hot Keys:** A single key (e.g., a celebrity profile) hitting one shard can overwhelm a node. *Solution: Local/Client-side caching or data replication across shards.*
  * **Cache Stampede (Thundering Herd):** Many requests hitting a miss simultaneously, overloading the DB. *Solution: Use "Promise" coalescing or jitter in TTLs.*
* **Comparisons:**
  * **vs. Memcached:** Pick Redis for data structures (Hashes/Sets) and persistence; pick Memcached for simpler, multithreaded key-value needs.
  * **vs. Local Cache (Guava/In-memory):** Pick Redis for consistency across multiple app nodes; pick Local Cache for ultra-low latency without network overhead.

## 6. The "Drop-In" Interview Script
> **Proposing it:** "We can introduce Redis here because we need to optimize for sub-millisecond read latency and offload the heavy read pressure from our relational database."
> **Justifying a feature:** "To handle memory management, we'll implement a TTL-based eviction policy combined with LRU, ensuring our most relevant data stays in-memory while preventing OOM (Out of Memory) errors."
> **Owning the trade-off:** "We accept eventual consistency here; if the DB update succeeds but the Redis invalidation fails, the data will remain stale until the TTL expires, which is acceptable for this product's requirements."

---

## 7. One-Minute Recap
* **Use when:** You need high-speed reads and want to shield your primary database from massive traffic.
* **Do NOT use when:** You need complex relational queries or your dataset is massive and lacks a clear 'hot' access pattern.
* **Key strength:** Blazing fast performance and versatile data structures.
* **Key weakness:** Limited by available RAM and susceptible to 'hot key' bottlenecks.
