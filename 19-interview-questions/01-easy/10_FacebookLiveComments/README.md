# Design Facebook Live Comments

## Problem Statement

Design a real-time comment system for Facebook Live Videos that enables millions of concurrent viewers to post and view comments on live video feeds with near-instantaneous delivery (<200ms end-to-end latency).

---

## 1. Requirements

### 1.1 Functional Requirements

| # | Requirement | Description |
|---|---|---|
| FR-1 | Post Comments | Viewers can post comments on a live video feed |
| FR-2 | View New Comments | Viewers can see new comments being posted in real-time while watching live video |
| FR-3 | View Comment History | Viewers can see comments made before they joined the live feed |

**Out of Scope:**
- Comment replies (threaded comments)
- Comment reactions (likes, emojis)
- Comment moderation/reporting
- Comment editing/deletion

### 1.2 Non-Functional Requirements (SPARCS)

| Dimension | Requirement | Target | Justification |
|-----------|---|---|---|
| **Scalability** | Support millions of concurrent videos & thousands of QPS per video | 2M concurrent live videos, 10K QPS per video | Facebook Live scale |
| **Performance (Latency)** | End-to-end comment delivery latency | <200ms (human perception threshold) | Real-time user experience |
| **Performance (Throughput)** | Comment ingestion throughput | 100K+ QPS (aggregate) | Peak comment load |
| **Availability** | System uptime | 99.95% | Tolerate regional outages, not all comments are critical |
| **Reliability** | Data durability | 99.99999% (7 nines) | Comments are user-generated content (permanent) |
| **Consistency** | Consistency model | **Eventual Consistency (CAP: Choose AP)** | Availability > Consistency; viewers accept out-of-order comments |
| **Security** | Authentication & Authorization | OAuth2 / JWT + Rate Limiting | Only authenticated users can post; prevent spam/abuse |

**CAP Theorem Trade-off:** We choose **Availability & Partition Tolerance** over Consistency. If a network partition occurs, we permit some viewers to miss comments temporarily but keep the system writing comments (eventual sync when partition heals).

---

## 2. Capacity Estimation

### 2.1 Assumptions

| Metric | Value | Rationale |
|--------|-------|-----------|
| Monthly Active Users (MAU) | 2.9 Billion | Facebook scale |
| Daily Active Users (DAU) watching Live | 500 Million | ~17% of MAU |
| Peak Concurrent Live Videos | 2 Million | Industry data + estimates |
| Avg Comments per Video | 5,000/hour | Range: 100–50K+ for viral content |
| Average Comment Length | 150 bytes | Typical text message |
| Peak QPS (aggregate) | 100K writes/sec | 2M videos × 0.05 writes/sec/video |
| Peak QPS per Video | 50 writes/sec | Busiest live streams |

### 2.2 Read & Write QPS

| Operation | Formula | Calculation | QPS |
|-----------|---------|---|---|
| **Write QPS (Comments Posted)** | Avg comments/hour ÷ 3600 × peak video multiplier | (5,000 ÷ 3,600) × 2M videos | **~2.8M writes/sec theoretically** |
| **Write QPS (Practical)** | DAU × avg comments/day ÷ 86,400 × peak factor | 500M × 10 ÷ 86,400 × 2 | **~115K writes/sec** |
| **Read QPS (Comment Fetch)** | Peak video viewers × fetch rate | 2M videos × 100 concurrent viewers × 1 fetch/5s | **~40M reads/sec** |
| **Read QPS (Real-time Stream via SSE)** | Viewers connected to server | 500M concurrent connections × broadcast factor | **~500K broadcast events/sec** |

**Key:** We focus on ~100K sustained write QPS and design for 50–100K broadcasts/sec per server cluster.

### 2.3 Storage Estimation (5-Year Retention)

| Data | Formula | Calculation | Size |
|------|---------|---|---|
| **Comments Table** | DAU × avg comments/day × avg comment size × 365 × 5 years | 500M × 10 × 150B × 365 × 5 | **~2.74 PB** |
| **Metadata** (IDs, timestamps, userId) | DAU × avg comments/day × 100B × 365 × 5 | 500M × 10 × 100B × 365 × 5 | **~0.84 PB** |
| **Total Database Storage** | Comments + Metadata | 2.74 + 0.84 | **~3.6 PB** |
| **Replication Factor** | 3× across regions | 3.6 PB × 3 | **~10.8 PB** |
| **Indexes** (liveVideoId, createdAt, userId) | ~40% of base data | 3.6 PB × 0.4 | **~1.44 PB** |
| **Total w/ Replication & Indexes** | Replication + Indexes | 10.8 + 1.44 | **~12.24 PB** |

### 2.4 Bandwidth Estimation

| Path | Formula | Calculation | Bandwidth |
|------|---------|---|---|
| **Inbound (Write Path)** | Write QPS × avg comment size | 100K × 150B | **~15 Gbps** |
| **Outbound (Real-time Broadcast)** | Broadcast QPS × avg comment size | 100K broadcasts × 150B | **~15 Gbps** |
| **Outbound (Read Path / Paginated Fetch)** | Read QPS × avg response | 40M reads/sec ÷ 10K RPS × 2KB response | **~80 Gbps (peak)** |
| **Total Peak Bandwidth** | Inbound + Outbound | 15 + 15 + 80 | **~110 Gbps** |

---

## 3. Core Entities & Data Model

### 3.1 Core Entities

| Entity | Description | Attributes |
|--------|---|---|
| **User** | Commenter or viewer | userId, username, avatar, email |
| **Live Video** | Active broadcast (owned by separate team) | liveVideoId, creatorId, title, viewers_count |
| **Comment** | Posted message on a live video | commentId, liveVideoId, userId, message, createdAt, deleted |

### 3.2 Database Schema

| Table | Field | Type | Constraints & Description |
|-------|-------|------|---|
| **comments** | commentId | UUID / Snowflake ID | Primary Key; globally unique, sortable by timestamp |
| | liveVideoId | UUID | Foreign Key (partition key); ensures comments grouped by video |
| | userId | UUID | Foreign Key; references user who posted |
| | message | TEXT (max 1000 chars) | Comment content; sanitized to prevent XSS |
| | createdAt | TIMESTAMP | Sortable; enables cursor-based pagination |
| | updatedAt | TIMESTAMP | Soft-delete tracking |
| | deleted | BOOLEAN (default: false) | Soft delete flag; prevents hard deletes |
| | metadata | JSON | Extra: reactions count, report count, visibility flags |

### 3.3 Indexes

| Index | Columns | Purpose |
|-------|---------|---------|
| **Primary Index** | commentId | Unique lookup by comment |
| **Query Index 1** | (liveVideoId, createdAt DESC) | Fetch comments for a live video, sorted by recency |
| **Query Index 2** | (liveVideoId, commentId DESC) | Cursor-based pagination for comment history |
| **Query Index 3** | (userId, createdAt DESC) | Retrieve user's comment history |

### 3.4 Relationships

- **User 1:N Comment** — One user can post many comments.
- **Live Video 1:N Comment** — One live video can have many comments.
- **Primary Key Strategy:** Snowflake ID (distributed, time-sorted, decentralized generation) preferred over UUID for better locality and indexing efficiency in a distributed database.

---

## 4. API Design

### 4.1 Endpoints

| Method | Endpoint | Request | Response | Description |
|--------|----------|---------|----------|---|
| **POST** | `/api/v1/comments/:liveVideoId` | `{ "message": "Cool stream!" }` **Header:** `Authorization: Bearer <JWT>` | `{ "commentId": "1234567890", "userId": "user123", "message": "Cool stream!", "createdAt": "2025-12-20T22:30:00Z" }` | Post a new comment on a live video |
| **GET** | `/api/v1/comments/:liveVideoId?cursor=<commentId>&limit=20&sort=desc` | Query params: cursor, limit (max 100), sort (asc/desc) | `{ "comments": [...], "nextCursor": "xyz", "hasMore": true }` | Fetch paginated comment history |
| **GET** | `/api/v1/comments/:liveVideoId/stream` | **WebSocket or SSE upgrade** | SSE Stream: `data: { "commentId": "...", "message": "...", "userId": "...", "createdAt": "..." }` | Subscribe to real-time comment stream for a live video |
| **DELETE** | `/api/v1/comments/:commentId` | **Header:** `Authorization: Bearer <JWT>` | `{ "status": "deleted" }` | Soft-delete a comment (out of scope; listed for completeness) |

### 4.2 Request/Response Examples

**POST /api/v1/comments/:liveVideoId**

POST /api/v1/comments/video_abc123 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
"message": "That was epic!"
}

// Response (201 Created)
{
"commentId": "1608979200000001",
"liveVideoId": "video_abc123",
"userId": "user_456",
"message": "That was epic!",
"createdAt": "2025-12-20T22:30:00Z"
}

**GET /api/v1/comments/:liveVideoId**

GET /api/v1/comments/video_abc123?cursor=1608979100000000&limit=20&sort=desc HTTP/1.1
Authorization: Bearer <JWT>

// Response (200 OK)
{
"comments": [
{
"commentId": "1608979200000001",
"userId": "user_456",
"message": "That was epic!",
"createdAt": "2025-12-20T22:30:00Z"
},
{
"commentId": "1608979150000002",
"userId": "user_789",
"message": "Amazing stream!",
"createdAt": "2025-12-20T22:29:10Z"
}
],
"nextCursor": "1608979050000003",
"hasMore": true
}

**GET /api/v1/comments/:liveVideoId/stream (Server-Sent Events)**

GET /api/v1/comments/video_abc123/stream HTTP/1.1
Authorization: Bearer <JWT>

// Server response (200 OK, Content-Type: text/event-stream)
data: {"commentId":"1608979250000010","userId":"user_111","message":"WOW!","createdAt":"2025-12-20T22:31:30Z"}
data: {"commentId":"1608979260000011","userId":"user_222","message":"Brilliant!","createdAt":"2025-12-20T22:31:40Z"}

---

## 5. High-Level Architecture

![Live comments](../../Images/FbLiveComments.excalidraw.svg)

### 5.1 Architecture Components

| Component | Technology | Role |
|-----------|---|---|
| **Client (Web/Mobile)** | JavaScript / Native App | Commenter or Viewer; sends POST & subscribes to SSE |
| **Load Balancer (LB)** | NGINX / HAProxy / AWS ALB | Round-robin or least-connection routing; Layer 4/7 |
| **API Gateway** | Kong / AWS API Gateway | Rate limiting, authentication, request routing, logging |
| **Comment Management Service** | Node.js / Java / Go | Handles POST /comments and GET /comments endpoints; CPU-bound write logic |
| **Realtime Messaging Service** | Node.js (with SSE/WebSocket) | Manages SSE connections; broadcasts comments to connected viewers |
| **Pub/Sub Broker** | Apache Kafka / Redis Pub/Sub | Decouples write path from broadcast path; ensures all servers receive new comments |
| **Comments Database** | DynamoDB / Cassandra / PostgreSQL (sharded) | Persistent storage; optimized for time-series comment writes |
| **Cache Layer** | Redis / Memcached | Caches recent comments & user sessions; reduces DB load |
| **Coordinator Service** | ZooKeeper / etcd / Consul | Maintains liveVideoId → Realtime Server mapping for routing |
| **Message Queue** | Kafka / RabbitMQ | Async comment processing, logging, analytics pipeline |
| **Monitoring/Logging** | Prometheus / ELK / Datadog | Metrics: latency, error rate, QPS; Logs: request traces |

---

## 6. Data Flow

### 6.1 Write Path (Post Comment)

**Sequence:**

1. **Client Action** — Commenter drafts comment and clicks "Post"
2. **Authentication** — API Gateway validates JWT token; extracts userId
3. **Rate Limit Check** — API Gateway checks if user has exceeded rate limit (e.g., 10 comments/min)
4. **Validation** — Comment Management Service validates message (length, content, sanitization)
5. **Database Write** — Comment persisted to **Comments DB** (partition key: liveVideoId, sort key: commentId)
6. **Pub/Sub Publish** — Upon successful write, service publishes comment to **Kafka topic**: `live-comments.{liveVideoId}`
7. **Response** — Client receives comment confirmation (201 Created)
8. **Broadcast (Async)** — All Realtime Messaging Servers subscribed to the topic receive the comment via Kafka
9. **SSE Distribution** — Each Realtime Server sends comment to its connected viewers for that liveVideoId

**Synchronous vs. Asynchronous Handoffs:**
- Write to DB: **Synchronous** (must succeed before responding to client)
- Publish to Pub/Sub: **Synchronous** (ensures no comments missed, but adds ~10–20ms latency)
- SSE broadcast: **Asynchronous** (non-blocking; server maintains open connections)

### 6.2 Read Path (Fetch Comment History)

**Sequence:**

1. **Client Request** — Viewer fetches comment history with `GET /comments/:liveVideoId?cursor=xyz&limit=20`
2. **Cache Lookup** — Comment Management Service checks **Redis Cache** for recent comments
   - Hit: Return cached results (~1ms)
   - Miss: Query database
3. **Database Query** — Query **Comments DB** with composite index `(liveVideoId, createdAt DESC)`:

SELECT * FROM comments
WHERE liveVideoId = 'video_abc'
AND createdAt < cursor_timestamp
ORDER BY createdAt DESC
LIMIT 20

4. **Cache Update** — Results cached for 5 minutes
5. **Response** — Return paginated results with `nextCursor` for subsequent fetches

**Cursor-Based Pagination:** Uses keyset pagination (comment ID as cursor) to avoid offset inefficiency and ensure consistency across multiple fetches.

### 6.3 Real-time Broadcast Path (Stream Comments)

**Sequence:**

1. **SSE Connection** — Viewer connects to `GET /api/v1/comments/:liveVideoId/stream`
2. **Server Registration** — Realtime Messaging Server adds viewer connection to in-memory map: `{liveVideoId: [connection1, connection2, ...]}`
3. **Kafka Subscription** — Realtime Server (if not already) subscribes to Kafka topic `live-comments.{liveVideoId}`
4. **New Comment Posted** — When commenter posts, Kafka receives event
5. **Fanout to Viewers** — Realtime Server receives event from Kafka and broadcasts to all connected SSE clients for that liveVideoId
6. **Client Receives** — Viewer's browser receives SSE frame and updates UI in real-time

**Total Latency Breakdown:**
- Client to API Gateway: ~20ms (network + processing)
- DB write: ~5ms
- Kafka publish + broker acknowledge: ~10ms
- Kafka consumer receives: ~10ms
- SSE broadcast: ~5ms
- Network to client: ~20ms
- **Total: ~70ms** (well under 200ms target)

---

## 7. Deep Dives

### 7.1 Real-time Broadcasting via Pub/Sub

**Challenge:** How to broadcast a comment from one server to all viewers across multiple servers?

**Solution: Apache Kafka + SSE**

1. **Decoupling via Pub/Sub** — When a comment is written to the database, the Comment Management Service publishes it to a Kafka topic scoped per live video: `live-comments.{liveVideoId}`.

2. **Distributed Subscription** — Each Realtime Messaging Server subscribes to Kafka topics for all live videos it is currently serving (based on routing logic).

3. **Fan-out to Connected Clients** — Upon receiving a message from Kafka, the server broadcasts the comment via SSE to all in-memory connections for that video.

4. **Reliability & Ordering** — Kafka ensures:
   - **At-least-once delivery**: Comments are not lost (persisted to Kafka broker)
   - **Ordering**: Comments within a partition (video) maintain FIFO order
   - **Scalability**: Multiple consumer instances (Realtime Servers) can scale horizontally

**Trade-offs:**
- **Pro**: Decouples services; horizontal scaling; fault-tolerant
- **Con**: Additional latency (~10–20ms) due to Kafka broker coordination

**Alternative: Direct Redis Pub/Sub**
- Simpler, lower latency (~5ms), but less reliable (no persistence, in-memory only)
- Suitable for systems that can tolerate occasional lost messages

### 7.2 Scaling to Millions of Concurrent Viewers

**Challenge:** How to route millions of viewers to Realtime Servers such that viewers of the same live video connect to the same server (or a small set of servers)?

**Solution: Consistent Hashing + Coordinator Service**

**Problem Breakdown:**

Without coordination, viewers of Live Video A spread across all servers → each server must broadcast to all servers → O(N²) fan-out complexity.

**Approach: Sticky Routing via Consistent Hashing**

1. **Load Balancer Routing** — When a viewer initiates SSE connection to `GET /comments/:liveVideoId/stream`:
   - API Gateway extracts `liveVideoId` from the URL
   - Applies consistent hash: `server = hash(liveVideoId) mod num_servers`
   - Routes all viewers of the same video to the same server (or a small replica set)

2. **Server Registration** — Each Realtime Server registers itself with a **Coordinator Service** (ZooKeeper, etcd, Consul):
   - Server heartbeat: "I'm online and healthy"
   - Coordinator maintains live list of active servers
   - Coordinator updates consistent hash ring when servers join/leave

3. **Routing Lookup** — Load Balancer caches the consistent hash ring and updates it upon coordinator notifications (eventual consistency is fine here).

**Scalability & Benefits:**

- All viewers of video X → Server 1 (by consistent hash)
- All viewers of video Y → Server 2 (by consistent hash)
- Comment for video X published to Kafka → only Server 1 consumes it → broadcasts to its viewers
- **Result:** O(1) fanout per server, linear scaling

**Failure Handling:**

- **Server Failure:** Coordinator detects missing heartbeat, removes server from hash ring
- **In-flight Connections:** Affected viewers reconnect; load balancer routes to next healthy server in replica set
- **Kafka Rebalance:** Consumer group rebalances; new server takes over topic partitions

### 7.3 Sharding Strategy

**Challenge:** As comments accumulate (3.6 PB in 5 years), queries may slow down.

**Solution: Partition by liveVideoId**

**Partitioning Key:** `liveVideoId`

- **Rationale**: Comments for a live video are always queried together; partitioning by video ensures related data is co-located
- **Write Path**: When posting a comment, hash(liveVideoId) determines the partition shard
- **Read Path**: When fetching history, look up the same shard

**Sharding Formula:**

shard_id = hash(liveVideoId) % num_shards
Example: 256 shards
video_abc123 → shard 0
video_def456 → shard 42
video_abc123 (again) → shard 0 (consistent)

**Hot Shard Mitigation:**

- **Viral Videos**: If a video accumulates 1M comments/hour (99th percentile), writes to that shard spike
- **Mitigation Strategies**:
   1. **Secondary Indexes**: Shard by (liveVideoId, timestamp) to spread writes across time-based buckets
   2. **Write Buffer**: Use in-memory buffer before flushing to DB (trade: slight consistency delay)
   3. **Dedicated Shard**: Assign viral videos to dedicated, over-provisioned shards
   4. **Queue Throttling**: Rate-limit writes at the application layer during extreme traffic

### 7.4 Caching Strategy

**Cache Placement:** Redis, co-located with Comment Management Service

**Caching Pattern: Cache-Aside (Lazy Loading)**

1. **Read Request** — On `GET /comments/:liveVideoId?cursor=xyz`:
   - Check Redis: `comments:{liveVideoId}:{cursor}:{limit}`
   - **Hit**: Return cached result (~1–2ms)
   - **Miss**: Query DB, populate cache, return result

2. **Write Path** — On `POST /comments/:liveVideoId`:
   - Write to DB first (synchronous)
   - Invalidate cache key: `DEL comments:{liveVideoId}:*` (wildcard invalidation)
   - Next read will refill cache

**Cache Expiration:** 5 minutes TTL (trade-off: stale data vs. cache efficiency)

**Hot Comments Cache:**

- **Most Recent 100 Comments**: Cache per live video for faster pagination start
- **TTL:** 30 seconds (rotates quickly as new comments arrive)
- **Size:** 100 comments × 1 KB = 100 KB per video; with 1M active videos = 100 GB (feasible)

**Cache Warming (Optional):**

- Pre-load top 100 comments for top 1,000 live videos during off-peak hours
- Reduces cold-start latency for popular streams

### 7.5 Consistency Model & Eventual Consistency

**Challenge:** In a distributed system with Kafka and multiple Realtime Servers, viewers may see comments out-of-order or with slight delays.

**Solution: Accept Eventual Consistency**

**Rationale:**
- Comments are inherently social; exact ordering is less critical than availability
- Users expect "near real-time" not "instant atomicity"
- Availability & Partition Tolerance (CAP AP) suits live chat better than strict consistency

**Guarantees Provided:**

1. **Causal Consistency (per user)**: If Alice posts comment X, then comment Y, Alice will see X before Y
   - Achieved via Kafka ordering within a partition
2. **FIFO Ordering (per video)**: Comments within a live video maintain order
   - Achieved via single Kafka partition per liveVideoId
3. **No Lost Comments**: Once written to DB, comments persist
   - Achieved via durable storage (DynamoDB, Cassandra, or PostgreSQL replication)

**Anomalies Tolerated:**

- Viewer A sees comment before Viewer B (network lag)
- Viewer C briefly sees comments out-of-order during server recovery (OK, self-heals)

### 7.6 Failure Modes & Recovery

| Failure Scenario | Impact | Mitigation |
|---|---|---|
| **Realtime Server Crashes** | Viewers lose real-time stream; but can still fetch history via GET endpoint | Multi-replica servers; load balancer routes to healthy replica; viewer reconnects automatically |
| **Kafka Broker Failure** | Comments not published until broker recovers | 3-way replication of Kafka cluster; automatic failover |
| **DB Partition Failure** | Comments may be temporarily unavailable | Multi-region replication; read replicas in standby; DNS failover |
| **Cache Miss (Stampede)** | Many concurrent readers query DB upon cache expiration | Implement probabilistic early expiration + cache warming; or request coalescing (wait for first result to populate cache) |
| **Comment Posted but SSE Not Broadcast** | Comment in DB but not seen by real-time viewers; resolved on refresh | Rare; monitor Kafka lag; ensure all servers consume topics; viewer can refresh page |

**Monitoring & Alerting:**

- **Latency**: p50, p99 end-to-end latency (target: <200ms)
- **Error Rate**: Failed write/read requests (target: <0.1%)
- **Kafka Lag**: Consumer lag per topic (alert if >5 seconds)
- **Cache Hit Rate**: % of cache hits (target: >80%)
- **DB CPU/Memory**: Per-shard resource utilization (alert if >80%)

### 7.7 Security

**Authentication:**

- **JWT or OAuth2** — Validate token before accepting POST request
- **Token Expiration** — 1 hour; require refresh for long sessions
- **User Identification** — Extract userId from token; associate with comment

**Authorization:**

- **Only authenticated users can post** — Verified at API Gateway
- **Rate Limiting** — Per-user limits (e.g., 10 comments/min, 100/hour)
   - Implemented at API Gateway or via Token Bucket algorithm
   - Prevents spam and abuse

**Input Validation & Sanitization:**

- **Message Length**: Max 1000 characters; reject oversized inputs
- **XSS Prevention**: Sanitize HTML/scripts in comment text before storage
   - Use DOMPurify or equivalent library
   - Store sanitized text; render as plain text on client
- **SQL Injection**: Use parameterized queries; ORM mitigates risk

**Data Privacy:**

- **Encryption in Transit**: HTTPS/TLS for all API calls
- **Encryption at Rest**: DynamoDB encryption enabled; Kafka broker SSL
- **PII Redaction**: Logs should not contain full message text; hash or truncate for debugging

---

## 8. Alternative Approaches & Trade-offs

### 8.1 Polling vs. Push (Real-time Delivery)

| Approach | Latency | Scalability | Complexity | Bandwidth |
|----------|---------|---|---|---|
| **Polling (GET every N sec)** | 1–30s | Poor (linear with viewers) | Low | High (redundant reqs) |
| **Server-Sent Events (SSE)** | <200ms | Good (fan-out via Kafka) | Medium | Low (one connection) |
| **WebSocket** | <200ms | Good (persistent connection) | Medium | Low (bidirectional) |

**Chosen: SSE + Kafka** — Simpler than WebSocket, sufficient for broadcast use case, native HTTP support.

### 8.2 Database Choice: DynamoDB vs. Cassandra vs. PostgreSQL

| Database | Write Latency | Scalability | Cost | Consistency | Best For |
|----------|---|---|---|---|---|
| **DynamoDB** | <5ms | Infinite (managed) | ~$1 per M writes | Eventual | Managed service, AWS ecosystem |
| **Cassandra** | <5ms | Linear (manual sharding) | Lower (self-hosted) | Tunable | High-volume distributed systems |
| **PostgreSQL (sharded)** | ~10ms | Linear (manual sharding) | Lower (self-hosted) | Strong | Complex queries, ACID guarantees |

**Chosen: DynamoDB** — Managed service, auto-scaling, no operational overhead; acceptable eventual consistency.

### 8.3 Pub/Sub: Kafka vs. Redis Pub/Sub vs. RabbitMQ

| Broker | Latency | Persistence | Scalability | Use Case |
|--------|---------|---|---|---|
| **Kafka** | ~10ms | ✓ (durable log) | ✓ (partitions) | High-throughput, replay capability |
| **Redis Pub/Sub** | ~5ms | ✗ (in-memory) | Limited | Low-latency but no durability |
| **RabbitMQ** | ~5ms | ✓ (with persistence) | Limited | Traditional message queuing |

**Chosen: Kafka** — Durability, partitioning per liveVideoId, easy to replay for recovery.

---

## 9. References

### Technical Resources
- **Deep Dive**: [Design Facebook Live Comments - HelloInterview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/fb-live-comments)
- **Video Walkthrough**: [YouTube - FB Live Comments Design](https://www.youtube.com/watch?v=LjLx0fCd1k8)

### Key Technologies
- **Pub/Sub**: Apache Kafka
- **Real-time Protocol**: Server-Sent Events (SSE)
- **Database**: DynamoDB or Cassandra
- **Cache**: Redis
- **Coordination**: ZooKeeper / etcd
- **Load Balancer**: NGINX, HAProxy, or AWS ALB

### Further Reading
- CAP Theorem: [Brewer's Theorem](https://en.wikipedia.org/wiki/CAP_theorem)
- Consistent Hashing: [Karger et al. (1997)](https://www.akamai.com/us/en/multimedia/documents/technical-publication/consistent-hashing-and-random-trees-distributed-caching-protocols-for-relieving-hot-spots-on-the-world-wide-web-technical-publication.pdf)
- Snowflake ID Generation: [Twitter Snowflake](https://github.com/twitter-archive/snowflake)

---

## 10. Document Metadata

| Field | Value |
|-------|-------|
| **Problem** | Design Facebook Live Comments |
| **Date** | December 21, 2025 |
| **Architecture Level** | High-Level Design (HLD) |
| **Target Audience** | Senior+ Engineers, System Design Interview Candidates |
| **Status** | Final |

---