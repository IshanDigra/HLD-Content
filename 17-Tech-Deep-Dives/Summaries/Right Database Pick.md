# System Design Comparison: PostgreSQL vs MongoDB vs Cassandra vs DynamoDB vs Redis vs Neo4j

---

## 1. The Decision Matrix (TL;DR)
| Feature | PostgreSQL | MongoDB | Cassandra | DynamoDB | Redis | Neo4j |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Primary Use Case** | **The Default Choice** / General Purpose | Schema Flexibility / Rapid Iteration | Massive Write Throughput | Fully-Managed / Serverless Scale | Sub-millisecond Caching / Transient Data | Complex Relationships / Traversals |
| **Mental Model** | Spreadsheets connected by pointers | Nested JSON folders | Distributed, append-only sparse matrices | Distributed hash map with B-tree sorted buckets | Giant in-memory hash map | Webs of nodes and connected edges |
| **Key Strength** | ACID guarantees & data integrity | Data locality for nested entities | Seamless horizontal write scaling with no single point of failure | Zero-ops management with built-in CDC (Streams) and caching (DAX) | Blazing fast sub-millisecond latency | $O(1)$ relationship traversals |
| **Key Weakness** | Difficult to scale writes horizontally | Denormalization leads to data drift | Rigid access patterns; deletions are expensive (Tombstones) | Vendor lock-in (AWS) and expensive at extreme write volume | Dataset size is bound by RAM | Horizontally partitioning a graph is notoriously hard |

## 2. Core Mechanics & Architecture
*Compare the underlying "how it works" across these categories:*

* **Data Model & Storage:**
  * **PostgreSQL:** Row-oriented storage utilizing B-Tree indexing. Relies on a schema-on-write model.
  * **MongoDB:** Stores BSON documents. Groups related data together, utilizing B-Tree variants for indexing.
  * **Cassandra:** Implements a partitioned wide-column storage model. It uses an LSM-Tree (Log-Structured Merge Tree) architecture where writes go to an in-memory Memtable (and Commit Log for durability) before being flushed sequentially to immutable SSTables on disk.
  * **DynamoDB:** A schema-less key-value/document store. Data physical location is determined by a Partition Key (via consistent hashing), and data within that partition is organized in a B-Tree indexed by an optional Sort Key.
  * **Redis:** In-memory key-value store. Single-threaded operations eliminate race conditions for isolated commands.
  * **Neo4j:** Uses index-free adjacency. Every node maintains direct memory pointers to adjacent nodes.

* **Consistency & Durability:**
  * **PostgreSQL:** Strongly consistent. Uses Write-Ahead Logging (WAL) and two-phase locking.
  * **MongoDB:** CP (Consistent and Partition Tolerant) system by default via a primary node.
  * **Cassandra:** AP (Available and Partition Tolerant) system favoring eventual consistency. Offers "tunable consistency" (e.g., using `QUORUM` reads/writes where a majority $n/2+1$ of replicas must respond). Write conflicts are resolved via "last write wins" timestamps.
  * **DynamoDB:** Defaults to Eventual Consistency (served by any replica) for high availability, but supports Strong Consistency (reads routed directly to the partition leader).

* **Scaling Strategy:**
  * **PostgreSQL:** Vertical scaling; read replicas for read scaling.
  * **Cassandra:** Scales horizontally via consistent hashing. Data is partitioned across a ring of virtual nodes (vnodes) mapping to physical hardware.
  * **DynamoDB:** Highly scalable via auto-sharding and load balancing across AWS infrastructure. Offers Global Tables for real-time cross-region replication.
  * **Redis:** Redis Cluster utilizing hash slots.

* **Failure Handling:**
  * **PostgreSQL:** Master-slave replication requiring promotion.
  * **Cassandra:** Peer-to-peer "gossip" protocol ensures universal cluster knowledge. Uses a Phi Accrual Failure Detector to identify offline nodes. If a node is down, coordinator nodes use "hinted handoffs" to temporarily store write data until the node returns.
  * **DynamoDB:** Fully-managed quorum-based replication across multiple AWS Availability Zones.

## 3. Performance & Latency Profiles
* **Throughput:** Cassandra is heavily write-optimized; its append-only mechanics make it the winner for sustained, massive write throughput. DynamoDB handles massive scale effortlessly but bills via Read/Write Capacity Units (RCU/WCU), making continuous extreme throughput very expensive.
* **Latency:** Redis dominates. DynamoDB provides predictable single-digit millisecond latency, which drops to sub-millisecond levels for reads if the DAX (DynamoDB Accelerator) in-memory cache is enabled.
* **Resource Overhead:** Cassandra requires heavy JVM tuning. DynamoDB has zero operational overhead as a fully-managed service.

## 4. The Decision Tree (When to Choose What)
* **Pick PostgreSQL IF:**
  * **You are in a system design interview.** (Default choice for 90% of relational data needs).
* **Pick Cassandra IF:**
  * You need to ingest massive amounts of write-heavy data (e.g., IoT metrics, logging, Discord messages).
  * You can strictly define your query patterns upfront (query-driven data modeling).
* **Pick DynamoDB IF:**
  * You are building in the AWS ecosystem and want zero operational maintenance.
  * You need built-in Change Data Capture via DynamoDB Streams to trigger Lambda functions or sync to Elasticsearch.
* **Pick MongoDB IF:**
  * Your data structure is highly nested and fluid.
* **Pick Redis IF:**
  * You need a transient cache or rate-limiting datastore.

## 5. Senior-Level "Gotchas"
* **Cassandra Pitfalls:** * **Tombstones:** Deletions are treated as "removal updates" (tombstones). High deletion rates bloat the SSTables and severely degrade read performance until a compaction process runs.
  * **Hot Partitions:** If a single partition gets too large (e.g., a highly active Discord channel), performance plummets. Mitigation requires composite partition keys (like appending a time-based "bucket" to the key).
* **DynamoDB Pitfalls:**
  * **Cost Scaling:** At extreme write volumes (e.g., hundreds of thousands of writes/sec), provisioned throughput costs can be prohibitive.
  * **Index Overhead:** Relying too heavily on Global Secondary Indexes (GSIs) doubles your storage and write costs, as they are maintained as entirely separate physical partitions.
  * **Scan Operations:** Using a `Scan` operation reads the *entire* table, consuming massive amounts of read capacity units (RCUs) and must be avoided at scale.
* **PostgreSQL Pitfalls:** Connection pooling limits.

## 6. The "Interview Pivot" Script
*How to justify the choice during the design phase:*

> **The Choice:** "For our core transactional data, I'm defaulting to **PostgreSQL** due to its ACID guarantees. However, for our high-volume event logging service, I will use **Cassandra** because its LSM-Tree architecture is specifically optimized for massive write throughput."
> **Acknowledging the Trade-off:** "While **Cassandra** prevents us from doing complex ad-hoc JOINs, we can mitigate this by denormalizing our data at the application layer or utilizing Cassandra's Materialized Views to maintain alternative access patterns."
> **The Alternatives:** "If our team was fully bought into the AWS ecosystem and we wanted to minimize operational overhead, **DynamoDB** would be an excellent choice. However, given the projected continuous write volume of 100,000 events per second, DynamoDB's Write Capacity Units would become cost-prohibitive compared to hosting our own Cassandra cluster."

---

## 7. Deep-Dive Evaluation Criteria: Finding the Breaking Point
In a system design interview, selecting the right database comes down to finding the breaking point of your default choice (usually PostgreSQL). Here is how to evaluate the databases across five critical dimensions:

### 7.1. QPS (Queries Per Second) & Node Capabilities
* **PostgreSQL:** A well-tuned, vertically scaled node (e.g., 64 cores, 256GB RAM) on SSDs comfortably handles **10,000 to 20,000 QPS** for a mixed workload. For heavily cached, read-only queries utilizing connection poolers (like PgBouncer), a single node can push **50,000+ QPS**. *Breaking Point:* If write QPS consistently exceeds 15k, vertical scaling becomes prohibitively expensive.
* **Cassandra:** Built for extreme throughput. A single commodity node can handle **10,000 to 30,000 write requests per second**. Because of its leaderless architecture and consistent hashing, scaling is purely linear. Need 100,000 write QPS? Add more nodes to the ring.
* **DynamoDB:** Handled automatically via AWS auto-sharding. A single physical partition (shard) under the hood supports a hard limit of **1,000 Write Capacity Units (WCUs) and 3,000 Read Capacity Units (RCUs) per second**.

### 7.2. Read vs. Write Heavy Workloads
* **PostgreSQL (Read-Heavy / Balanced):** Excellent for read-heavy workloads where reads can be scaled horizontally via read replicas. However, because it uses a B-Tree index, high-volume random writes cause page fragmentation and disk I/O bottlenecks.
* **Cassandra (Write-Heavy):** The absolute king of write-heavy workloads (time-series, chat messages, logging). It utilizes an LSM-tree where writes are appended in memory (Memtable) and sequentially flushed to disk (SSTable), avoiding random disk seeks entirely.
* **DynamoDB (Mixed / Unpredictable):** Handles both exceptionally well but bills you per capacity unit. It handles massive throughput flawlessly, but extreme write volumes can become a massive financial cost.

### 7.3. Capacity Estimation (Data Size)
* **PostgreSQL:** The operational sweet spot is under **3–5 Terabytes**. While it can technically hold much more, tasks like vacuuming, index rebuilding, and taking backups become incredibly slow and risky at larger scales.
* **Cassandra:** Designed for **Petabyte-scale** storage. It handles massive, infinitely growing datasets effortlessly (e.g., Discord storing trillions of messages by appending a time-based "bucket" to partition keys).
* **DynamoDB:** Infinite storage capacity, fully managed. However, individual items have a strict size limit of **400KB**. If you need to store large blobs, you must store the blob in S3 and save the URL reference in DynamoDB.

### 7.4. Transactional & Speed Requirements
* **PostgreSQL (Strict Consistency):** Provides strict ACID guarantees. For financial transactions, inventory counts, or billing, Postgres is mandatory. It ensures data is never left in a partially updated state via two-phase locking and Write-Ahead Logs. Latency is typically in the low milliseconds.
* **DynamoDB (Speed & Flexibility):** Delivers predictable, single-digit millisecond latencies. Enabling DAX (DynamoDB Accelerator) drops read latency to **sub-millisecond** levels. Defaults to Eventual Consistency for high availability, but can be configured for Strong Consistency and fully supports ACID transactions.
* **Cassandra (Availability):** Does *not* offer ACID transactions. It is highly available and eventually consistent, resolving conflicts via "last write wins" timestamps. Fast, but sacrifices strict data correctness under network partitions.

### 7.5. Relationship Between the Data
* **PostgreSQL (Complex Relationships):** If entities have many-to-many relationships and exact query patterns are unknown upfront, choose Postgres. Its relational model (Foreign Keys, JOINs, secondary indexes) allows ultimate query flexibility.
* **DynamoDB (Hierarchical / Key-Value):** Schema-less and flexible, but cannot perform JOINs. Data modeling is entirely query-driven. Querying by attributes other than the primary key requires duplicating data into Global Secondary Indexes (GSIs), incurring extra storage and write costs.
* **Cassandra (Flat / Time-Series):** No JOINs, no referential integrity, and heavy denormalization is required. The schema must be designed entirely based on application access patterns upfront.

---

## 8. Quick Summary Table
| Constraint | Winner | Why? |
| :--- | :--- | :--- |
| **System Design Default** | PostgreSQL | Handles 95% of use cases gracefully; standard for data integrity. |
| **Zero-Ops / Serverless** | DynamoDB | Fully managed by AWS with built-in caching (DAX) and CDC (Streams). |
| **Highest Write Throughput** | Cassandra | LSM-Trees and leaderless architecture scale linearly with zero downtime. |
| **Lowest Latency** | Redis | In-memory data structures bypass disk I/O entirely. |
| **Ease of Operations** | DynamoDB | AWS handles all patching, scaling, and node management. |
