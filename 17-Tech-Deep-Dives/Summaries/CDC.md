# System Design Quick-Read: Change Data Capture (CDC)

---

## 1. The 1-Minute Pitch
* **What it is:** A design pattern used to track and capture database changes (inserts, updates, deletes) in real time and stream them to downstream systems[cite: 10].
* **Mental Model:** Think of it as a "Database Observer" that broadcasts every data mutation as an event to keep the rest of the system in sync.
* **System Placement:** Sits between the primary Source Database and downstream consumers like search engines, caches, and data lakes[cite: 8, 19].
* **When to think of it:** * **Interview Trigger 1:** "How do we keep a search index (Elasticsearch) or cache (Redis) consistent with our primary DB without manual dual-writes?"[cite: 8, 20].
  * **Interview Trigger 2:** "We need to move data from a transactional DB to an analytics warehouse (Snowflake) with near-zero latency"[cite: 123].

## 2. Core Fundamentals Cheat Sheet
* **Data Model:** Event-driven stream of row-level change records (e.g., JSON or Avro payloads)[cite: 25, 41].
* **Consistency:** Provides **eventual consistency** for downstream systems[cite: 20].
* **Durability:** High; typically relies on the database's Write-Ahead Log (WAL) and persistent message queues like Kafka[cite: 76, 156].
* **Scaling Model:** Scales by partitioning the underlying message stream (e.g., Kafka topics) and leveraging low-impact log reading[cite: 85, 92].
* **Failure Model:** Log-based CDC handles restarts gracefully by resuming from the last read log offset, ensuring no data loss[cite: 86, 188].
* **Latency Profile:** Near real-time; typically sub-second or low milliseconds from DB commit to consumer visibility[cite: 87, 156].

## 3. Architecture & Mechanics
**3.1 The Write Path**
* Client performs a DML operation (Insert/Update/Delete) on the Source Database[cite: 10].
* For Log-based CDC: The DB writes to its internal **Write-Ahead Log (WAL)** or Binlog[cite: 76].
* The **CDC Engine** (e.g., Debezium) intercepts these low-level log entries without blocking the original transaction[cite: 77, 155].

**3.2 The Read Path**
* The CDC Engine extracts the change event, including "before" and "after" states[cite: 40].
* The event is published to a **Message Queue** (e.g., Kafka)[cite: 33, 41].
* **Downstream Applications** (Consumers) subscribe to the stream and apply changes to their own local state (e.g., updating a cache or index)[cite: 37, 41].

**3.3 Scaling & High Availability**
* **Log-Based Efficiency:** Minimizes primary DB impact by reading existing logs rather than querying tables[cite: 83].
* **Distributed Streaming:** Integration with Kafka allows for high throughput and horizontal scaling of consumers[cite: 80, 156].

**3.4 Operational Knobs**
* **Inclusion Lists:** Filtering which specific databases or tables are monitored[cite: 165, 177].
* **Snapshotting:** The ability to perform an initial dump of existing data before starting the real-time stream.
* **Ordering Guarantees:** Critical for maintaining data integrity across distributed services[cite: 191, 192].

## 4. Interview Use Cases
* **Common Patterns:** Microservices communication (outbox pattern), real-time analytics, cache invalidation, and audit logging[cite: 93, 107, 123, 139].
* **When to CHOOSE it:** * Need for high efficiency and minimal impact on the primary database[cite: 83].
  * Requiring a comprehensive capture of all changes, including deletes[cite: 86].
  * Moving away from slow, stale batch ETL jobs[cite: 9].
* **When to AVOID it:** * Small projects where simple timestamp-based polling is "good enough"[cite: 53].
  * When the database does not expose its transaction logs in a usable way[cite: 90].
  * If the system cannot handle the added complexity of managing a streaming infrastructure[cite: 91].

## 5. Trade-offs, Pitfalls, & Alternatives
* **Common Gotchas (The "Senior" Signals):**
  * **Schema Evolution:** If a DB column is added or dropped, the CDC pipeline must adapt gracefully to prevent downstream crashes[cite: 187, 188].
  * **Event Ordering:** In distributed systems, updates must be processed in the exact sequence they occurred at the source to avoid state corruption[cite: 191, 192].
  * **Sensitive Data:** Transaction logs often contain raw data; encryption and masking are vital for compliance[cite: 193, 194].
* **Comparisons:**
  * **vs. Dual Writes:** CDC is superior because it avoids the "partial success" problem (where DB update succeeds but the second write to the cache fails).
  * **vs. Trigger-Based CDC:** Log-based is preferred because triggers add overhead to every transaction and slow down database writes[cite: 72, 92].

## 6. The "Drop-In" Interview Script
> **Proposing it:** "We can introduce **Log-based CDC** here because we need to keep our Elasticsearch search index synchronized with the primary PostgreSQL database in near real-time without impacting transaction performance[cite: 83, 87]."
> **Justifying a feature:** "To handle **cache invalidation**, CDC provides an automated way to trigger updates in Redis whenever the underlying record changes, preventing users from seeing stale content[cite: 139, 153]."
> **Owning the trade-off:** "We accept the **operational complexity** of running Kafka and Debezium because our product prioritizes near-zero data latency for analytics over the simplicity of batch jobs[cite: 9, 91]."

---

## 7. One-Minute Recap
* **Use when:** You need real-time sync across multiple systems (search, cache, warehouse) with minimal DB overhead[cite: 8, 83].
* **Do NOT use when:** Your database doesn't support log access or you have very simple, low-frequency data movement needs[cite: 53, 90].
* **Key strength:** High-efficiency, real-time capture of all change types (including deletes)[cite: 86, 87].
* **Key weakness:** High architectural complexity and dependency on specific database internal features[cite: 89, 91].