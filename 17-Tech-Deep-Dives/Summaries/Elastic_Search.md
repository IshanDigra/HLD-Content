# System Design Quick-Read: Elasticsearch

***

## 1. The 1-Minute Pitch
* **What it is:** Distributed search and analytics engine built on top of Apache Lucene, optimized for full-text search, filtering, and ranking over JSON documents.[1][2]
* **Mental Model:** Think of it as a denormalized, search-optimized secondary index that sits next to your primary OLTP database.[1]
* **System Placement:** Typically runs as a separate search cluster; application writes authoritative data to a primary store (Postgres, DynamoDB, etc.) and streams changes into Elasticsearch for fast read/search workloads.[1]
* **When to think of it:**
    * When the interview problem needs rich search: full-text, relevance ranking, filtering, faceting, or complex sorting over large datasets.[1]
    * When you must scale read-heavy query traffic beyond what a single database’s full-text index can handle.[1]

## 2. Core Fundamentals Cheat Sheet
* **Data Model:** Document store (JSON documents) grouped into indices; internally implemented via Lucene inverted indices and columnar doc values for fast search, sort, and aggregations.[1]
* **Consistency:** Per-document write consistency within a replication group, but search visibility is *near real-time* and thus eventually consistent; new writes become searchable after a refresh (≈ 1s by default) or explicit refresh/wait-for options.[1][2][3]
* **Durability:** Writes go to an in-memory buffer and a translog (write-ahead log) that is fsync’d either per request (`request`) or periodically (`async`), then flushed into immutable Lucene segments.[1][4]
* **Scaling Model:** Horizontal scale via index-level sharding across data nodes and replication of each shard; coordinating nodes fan out queries to all relevant shard copies and merge results.[1][5]
* **Failure Model:** Master-eligible nodes run leader election and manage shard placement; primary shard loss triggers promotion of replicas; under network partitions, Elasticsearch generally favors availability of reads/writes over strict global consistency.[1][6][7]
* **Latency Profile:** Indexing latency is typically low milliseconds for acknowledgement, with search visibility lagging by the refresh interval; query latency is usually low tens of milliseconds when data and indices fit in memory.[1][2]

## 3. Architecture & Mechanics

**3.1 The Write Path**
* Client sends an index/update request to any node, which acts as a coordinating node and routes the operation to the primary shard in the target index’s replication group (routing typically based on document ID or a custom routing key).[1][5]
* The primary shard appends the operation to the translog, updates in-memory structures, forwards to replica shards, and acknowledges once the required number of shard copies have applied the change; a later refresh turns buffered changes into new immutable Lucene segments that become searchable.[1][4][5]
* Updates are implemented as “delete + reinsert”: the old version is marked deleted in segments and a new document version is indexed; optimistic concurrency control uses a `_version` field so clients can avoid overwriting concurrent updates.[1]

**3.2 The Read Path**
* Client issues a search to a coordinating node, which parses the query, identifies target shards (all primaries and/or replicas for the index), and sends parallel search requests to each shard copy in the replication group.[1][5]
* Each shard uses its Lucene inverted index to find candidate documents, uses doc values for sorting/aggregations, and returns its top-k hits plus metadata; the coordinating node merges, re-sorts globally, and returns a single ranked result set.[1]
* Pagination can be done with simple `from`/`size` (cheap for shallow pages) or cursor-style `search_after`, optionally combined with Point-In-Time (PIT) to get a consistent snapshot across pages without being affected by ongoing writes.[1]

**3.3 Scaling & High Availability**
* **Routing/Sharding:** Each index is configured with a fixed number of primary shards; documents are assigned to a primary shard via a hash of the routing key (by default the document ID), which distributes data and query load across nodes.[1][5]
* **Replication:** Each primary shard has one or more replicas on other nodes; replicas provide high availability and also participate in serving read traffic, letting you scale search QPS roughly with the number of shard copies.[1][5]
* **Tiered Data Nodes:** Data nodes can be configured as hot/warm/cold/frozen tiers so frequently queried, mutable data resides on fast nodes, and older or rarely accessed indices sit on cheaper, slower storage.[1]

**3.4 Operational Knobs**
* **Refresh & Visibility:** The `refresh_interval` controls how often new writes become visible to search; `?refresh=wait_for` and explicit `POST _refresh` calls can force newer data to appear at the cost of write throughput and latency.[1][2][3]
* **Durability Tuning:** `index.translog.durability=request` fsyncs after every write for strong durability; `async` batches fsyncs (e.g., every 5s) for higher throughput but risks losing the latest operations on crash.[4]
* **Replication & Consistency:** Settings such as `wait_for_active_shards`, write consistency modes, and search preferences (`_primary`, `_local`, etc.) let you trade off strictness of shard participation vs availability and latency.[8][9]
* **Data Layout:** Mappings and field types determine which fields are indexed, how they’re analyzed, and which are stored as doc values; over-indexing unused fields inflates heap and segment size, hurting performance.[1]

## 4. Interview Use Cases
* **Common Patterns:** Product and document search with filtering and ranking; log/event search and analytics; user-generated content search (posts, comments, reviews); faceted browse pages with counts per category; geo or time-based search over large datasets.[1]
* **When to CHOOSE it:**
    * When you need full-text search, relevance scoring, sorting, and aggregations over large, evolving datasets with high read QPS.[1][2]
    * When primary OLTP databases (even with full-text indices) can’t handle search complexity or scale, and you are comfortable with near real-time rather than strictly up-to-date results.[1][3]
* **When to AVOID it:**
    * When the workload is highly write-heavy with frequent in-place updates or counters (e.g., per-click metrics) where immutable-segment semantics and merges become a bottleneck.[1]
    * When the system requires strict ACID multi-row transactions or must always read the latest committed value (e.g., financial ledger, inventory correctness) rather than tolerate stale search results.[1][3]
    * When the dataset is small and static enough that a single database with an index or simple in-memory search suffices; Elasticsearch adds operational overhead without clear benefit in this regime.[1]

## 5. Trade-offs, Pitfalls, & Alternatives
* **Common Gotchas (The "Senior" Signals):**
    * **Data Correctness:** Search is eventually consistent: writes may be acknowledged but not yet visible to queries until after a refresh; CDC or sync pipelines drifting from the source-of-truth database can leave the index out of sync.[1][2][10]
    * **Data Correctness:** Updates overwrite whole documents unless you use partial `_update` plus version checks, so naive upserts can lose concurrent changes; heavy update/delete traffic causes many tombstones and forces costly segment merges.[1]
    * **Operational:** Hot shards from skewed routing keys or bad sharding (e.g., one shard handling most traffic) limit horizontal scalability even in a large cluster.[1]
    * **Operational:** Overly rich mappings and too many indexed fields (“field explosion”) increase heap usage and GC pressure, slow down queries, and extend recovery/merge times.[1][7]
    * **Operational:** Deep pagination with large `from` offsets forces Elasticsearch to sort and accumulate many hits that are thrown away, leading to slow queries and high memory usage; `search_after` + PIT is preferred for deep scroll-like access.[1]

* **Comparisons:**
    * **vs. Database Full-Text (e.g., Postgres index):** Elasticsearch shines when you need complex full-text relevance, aggregations, and horizontal scale over many nodes; a database index is simpler and strongly consistent but usually scales only vertically and offers less flexible scoring.[1][2]
    * **vs. Key-Value Cache (e.g., Redis):** Elasticsearch is for rich search and analytics across many documents, not simple key-based lookups; Redis offers sub-millisecond, strongly consistent reads/writes per key but doesn’t provide full-text search, ranking, or faceting by default.[1]

## 6. The "Drop-In" Interview Script
> **Proposing it:** "We can introduce Elasticsearch here because we need a horizontally scalable, search-optimized read path for full-text and filtered queries without overloading the primary database."
> **Justifying a feature:** "To handle high read QPS and rich query patterns, Elasticsearch partitions data into shards and replicas, uses inverted indices and doc values for fast search and aggregations, and lets us denormalize documents for single-hop queries."
> **Owning the trade-off:** "We accept near real-time, eventually consistent search results and extra operational complexity because the product prioritizes fast, relevant search and high availability over perfectly up-to-date, transactional reads."

***

## 7. One-Minute Recap
* **Use when:** You need scalable, low-latency full-text search, filtering, and aggregations over large document sets, and can tolerate slightly stale results.[1][2]
* **Do NOT use when:** Your core workload is highly transactional, write-heavy, or requires strict read-your-writes consistency and complex relational joins.[1][3]
* **Key strength:** Combines Lucene’s powerful search primitives with distributed sharding and replication to deliver flexible, high-throughput search and analytics.[1]
* **Key weakness:** Operationally complex, not a primary source-of-truth database, and only offers near real-time, eventually consistent views of the underlying data.[1][3]