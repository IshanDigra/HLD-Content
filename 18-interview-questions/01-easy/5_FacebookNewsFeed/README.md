# Facebook News Feed System Design

## 1. Requirements

### 1.1 Functional Requirements

| ID | Requirement | Description |
|----|-------------|-------------|
| FR1 | Create Posts | Users should be able to create posts with content |
| FR2 | Follow Users | Users should be able to follow/friend other users (uni-directional) |
| FR3 | View Feed | Users should be able to view a feed of posts from people they follow in chronological order |
| FR4 | Pagination | Users should be able to page through their feed |

Below the Line (Out of Scope):  
Like and comment functionality on posts, private posts or restricted visibility, post editing or deletion, and rich media types are initially out of scope and can be added later as incremental features.

### 1.2 Non-Functional Requirements (SPARCS Framework)

| Attribute | Requirement | Target Metric |
|-----------|-------------|---------------|
| Scalability | Handle massive user base | 2 billion users |
| Performance | Low latency operations | < 500ms for posting and feed viewing |
| Availability | High availability system | 99.99% uptime, prioritize availability over consistency |
| Reliability | No data loss | Durable storage of posts and relationships |
| Consistency | Eventual consistency acceptable | Up to 1 minute of post staleness tolerated |
| Security | Rate limiting and authentication | Prevent abuse, secure user data |

From a CAP theorem perspective, this system prioritizes **Availability** and **Partition Tolerance** (AP system), accepting eventual consistency for feed updates and propagation of new posts to followers.

---

## 2. Capacity Estimation & Constraints

### 2.1 Traffic Estimates

| Metric | Calculation | Result |
|--------|-------------|--------|
| Daily Active Users (DAU) | Given | 2 billion users |
| Posts per user per day | Estimated average | 2 posts |
| Feed requests per user per day | Estimated average | 20 requests |
| Total posts per day | 2B DAU × 2 posts | 4 billion posts/day |
| Total feed requests per day | 2B DAU × 20 requests | 40 billion requests/day |
| Write QPS | 4B posts / 86,400 sec | ~46,300 posts/sec |
| Read QPS | 40B requests / 86,400 sec | ~463,000 requests/sec |
| Peak QPS (3x average) | multiplier on average | Write: ~139K/sec, Read: ~1.4M/sec |

### 2.2 Storage Estimates

Assume an average post is 1 KB including text and metadata.

| Component | Calculation | Result |
|-----------|-------------|--------|
| Average post size | Text content + metadata | 1 KB |
| Daily storage (posts) | 4B posts × 1 KB | 4 TB/day |
| Yearly storage (posts) | 4 TB × 365 | ~1.46 PB/year |
| 5-year storage (posts) | 1.46 PB × 5 | ~7.3 PB |
| Follow relationships | 2B users × 500 follows avg × 16 bytes | ~16 TB |
| Total 5-year storage | Posts + relationships + overhead (~20%) | ~9 PB |

### 2.3 Bandwidth Estimates

Assume each feed request returns on average 10 posts of 1 KB each.

| Direction | Calculation | Result |
|-----------|-------------|--------|
| Incoming (writes) | 4B posts × 1 KB / 86,400 sec | ~46 MB/sec (~370 Mbps) |
| Outgoing (reads) | 40B requests × 10 posts × 1 KB / 86,400 sec | ~4.6 GB/sec (~37 Gbps) |
| Peak bandwidth | 3x average | Write: ~1.1 Gbps, Read: ~111 Gbps |

### 2.4 Memory Estimates (Cache)

| Component | Calculation | Result |
|-----------|-------------|--------|
| Hot posts cache (80/20 rule) | 20% of daily posts × 1 KB | ~800 GB |
| Precomputed feeds (active users) | 200M active users × 200 posts × 1 KB | ~40 TB |
| Follow graph cache | 200M users × 500 follows × 16 bytes | ~1.6 TB |

---

## 3. Core Entities

### 3.1 Data Model Overview

The system tracks three primary entities.

The **User** entity represents individual application users, storing core identity and profile information. The **Post** entity encapsulates content authored by users, including text and associated metadata such as timestamps. The **Follow** entity represents uni-directional social relationships where one user follows another and is modeled explicitly to allow efficient querying of follower and following sets.

Additionally, a **Precomputed Feed** entity is introduced as a materialized view of the feed for a user, storing references to relevant posts for low-latency retrieval.

### 3.2 Database Schema

#### User Table

| Field Name | Type | Description/Constraint |
|------------|------|------------------------|
| userId | UUID | Primary Key, unique identifier |
| email | String | Unique, indexed |
| name | String | Display name |
| createdAt | Timestamp | Account creation time |

The primary key is chosen as a UUID to ensure global uniqueness and avoid sequential key hotspots or information leakage about user counts. Secondary indexes can be maintained on the email field to enforce uniqueness and enable login or lookup by email.

#### Post Table (DynamoDB Single Table Design)

| Field Name | Type | Description/Constraint |
|------------|------|------------------------|
| PK | String | Partition Key: typically `postId` for direct post lookup or `userId#post` pattern in a single-table design |
| SK | String | Sort Key: `metadata` or `createdAt` to support ordering |
| postId | UUID | Unique post identifier |
| userId | String | Creator of the post |
| content | String | Post content (max 5000 characters) |
| createdAt | Timestamp | Post creation time (used for ordering and cursor pagination) |

A Global Secondary Index (GSI) is defined to support fetch-by-user queries:

- GSI1  
  Partition Key: `userId`  
  Sort Key: `createdAt`  

This index enables efficiently retrieving a user's posts in reverse chronological order.

#### Follow Table (DynamoDB Single Table Design)

| Field Name | Type | Description/Constraint |
|------------|------|------------------------|
| PK | String | Partition Key: `userFollowing` (follower's userId) |
| SK | String | Sort Key: `userFollowed` (followee's userId) |
| createdAt | Timestamp | Timestamp when the follow relationship was created |

A GSI is defined to support efficient lookup of followers of a user:

- GSI1  
  Partition Key: `userFollowed`  
  Sort Key: `userFollowing`

This models the many-to-many user-follow relationships such that both "who I follow" and "who follows me" queries are efficient.

#### Precomputed Feed Table (DynamoDB)

| Field Name | Type | Description/Constraint |
|------------|------|------------------------|
| PK | String | Partition Key: `userId` (the feed owner) |
| SK | String | Sort Key: `timestamp#postId` for chronological ordering and uniqueness |
| postId | UUID | Reference to the Post entity |
| createdAt | Timestamp | Post creation time |
| isPrecomputed | Boolean | Flag indicating if this item came from the fan-out precomputation process |

Only the latest fixed number of posts (for example 200) per user are stored to keep partition sizes bounded and to avoid unbounded growth in feed materialization. Older items are fetched on demand via pull from followed users.

### 3.3 Relationships

The relationship between User and Post is naturally one-to-many; a single user can create many posts across time. This mapping is realized by associating each Post with a `userId` foreign key and leveraging GSI queries on `userId`.

The User to User relationship is many-to-many and is captured in the Follow table. A follower can follow many users, and any user can be followed by many others. The Follow table effectively acts as an adjacency list, with different access patterns supported via primary key and GSI.

The User to Precomputed Feed mapping is also one-to-many, with each user owning many feed entries. This feed can be viewed as a denormalized materialization of relevant posts, helping optimize read performance.

The choice of primary keys is driven by access patterns. UUIDs are preferred for `userId` and `postId` to avoid hotspots and maintain independence across partitions. Composite keys of the form `timestamp#postId` for sort keys allow chronological ordering while still disambiguating posts created at the same second.

NoSQL, such as DynamoDB, is preferred over a relational database for this workload because it supports massive horizontal scalability, is optimized for key-value and time-series access patterns, and eliminates the need for complex manual sharding that a relational system would require to handle tens of thousands of writes and hundreds of thousands of reads per second.

---

## 4. API Design

### 4.1 API Endpoints

| Method | Endpoint | Request Params / Body | Response | Description |
|--------|----------|-----------------------|----------|-------------|
| POST | `/posts` | Body: `{ content: string }` | `{ postId: string, ... }` | Create a new post |
| PUT | `/users/{id}/followers` | Body: `{ followerId: string }` | `200 OK` | Follow a user |
| GET | `/feed` | Query: `pageSize: int, cursor?: timestamp` | `{ items: Post[], nextCursor: string }` | Get authenticated user's feed |
| GET | `/users/{id}/posts` | Query: `pageSize: int, cursor?: timestamp` | `{ items: Post[], nextCursor: string }` | Get posts for a specific user |
| GET | `/users/{id}/following` | Query: `pageSize: int, cursor?: string` | `{ items: User[], nextCursor: string }` | Get list of users this user follows |
| GET | `/users/{id}/followers` | Query: `pageSize: int, cursor?: string` | `{ items: User[], nextCursor: string }` | Get list of followers for this user |

### 4.2 API Details

#### POST /posts

Request:

```json
{
  "content": "This is my post content"
}
```

Response:

```json
{
  "postId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "user123",
  "content": "This is my post content",
  "createdAt": "2024-12-02T10:30:00Z"
}
```

This endpoint validates authentication, enforces content length constraints, and persists the new Post record. The creation triggers an asynchronous fan-out process via a message queue but the HTTP request completes as soon as the post is durably stored.

#### GET /feed

Request parameters:

- `pageSize` (required): Number of posts to return, with an upper bound (for example 100).
- `cursor` (optional): A timestamp or encoded cursor string representing the point after which to fetch additional posts for pagination.

Sample response:

```json
{
  "items": [
    {
      "postId": "post123",
      "userId": "user456",
      "content": "Post content",
      "createdAt": "2024-12-02T10:30:00Z"
    }
  ],
  "nextCursor": "2024-12-02T10:29:00Z"
}
```

Cursor-based pagination relies on the `createdAt` timestamp and `postId` ordering to ensure stability even when new posts are created between page fetches.

---

## 5. Data Flow

### 5.1 Write Path (Post Creation)

The write path begins at the client where a user composes a post and taps the submit button. The mobile or web client packages the content along with authentication credentials (a JWT access token) and sends an HTTPS POST request to the `/posts` endpoint. This request first encounters the API Gateway or edge layer, which verifies the JWT, applies rate-limiting rules, and routes the request to the Post Service.

The Post Service performs validation checks such as ensuring the content length is within acceptable bounds, detecting illegal or malformed content, and tracing the request for observability. It generates a unique `postId` (typically a UUID) and writes the post record into the Post Table in DynamoDB, using a partition key strategy suited for high write throughput. Once DynamoDB acknowledges the write as durable, the service returns a success response to the client with the new `postId` and metadata such as `createdAt`.

In parallel, as part of the same request handling or immediately afterward, the Post Service publishes an event to an asynchronous Fan-Out Queue (for example, SQS or Kafka). This event contains the `postId`, `userId`, and timestamp. Crucially, the HTTP request response does not wait for fan-out to complete; the queue acts as a buffer and decouples the synchronous user path from downstream operations. This keeps the p99 write latency under the 500 ms target.

Async Worker processes continuously poll the Fan-Out Queue. For each new post event, a worker queries the Follow Table (typically via a GSI on `userFollowed`) to obtain all followers of the post author. If the follower count is below a configured celebrity threshold, and if the follower is considered active (for example, last login within 30 days), the worker creates or updates entries in the Precomputed Feed Table for each follower, appending the `postId` with the appropriate `timestamp#postId` sort key. This fan-out effectively materializes the feed for most users while leaving celebrity posts to be pulled on-demand. All of this fan-out work is asynchronous and resilient to spikes in traffic due to the buffering nature of the queue.

### 5.2 Read Path (Feed Viewing)

The read path begins when a user launches the app or pulls-to-refresh their feed. The client issues a GET request to `/feed` with optional pagination parameters. As with writes, the request passes through the API Gateway, which authenticates the user and applies per-user and per-IP rate limits before routing to the Feed Service.

The Feed Service first attempts to serve the request from the Precomputed Feed Table by querying items with a partition key equal to the user's `userId`, and a sort key greater than the supplied cursor if provided. The service fetches the most recent entries up to the requested `pageSize`. For each entry, it needs the underlying Post content, so it queries the Post cache layer, typically a Redis cluster. If the cache returns a hit for the given `postId`, the content is immediately used; otherwise, the service fetches the post from DynamoDB and populates the cache for subsequent requests, following a cache-aside strategy.

If the precomputed feed does not have enough posts to satisfy the page size, perhaps because many of the accounts the user follows are marked as celebrities for which feeds are not precomputed, the Feed Service supplements from a pull path. It obtains the list of followed users via the Follow Service and filters for celebrity users. For each celebrity, it queries the Post Table's GSI to fetch recent posts, merges these on-demand posts with any precomputed entries already fetched, and sorts the combined list by timestamp in descending order. It then truncates to the requested page size and constructs a new `nextCursor` based on the timestamp (and possibly `postId`) of the last post returned.

The Feed Service finally returns the assembled feed items to the client. Because most users' feeds are heavily served from precomputed data and cached posts, the median and p95 latencies are kept very low. For users who follow many celebrities, slight additional latency is expected due to the extra Post Table lookups, but the system design still targets sub-500 ms end-to-end latency through efficient caching and limited pull depth.

---

## 6. High-Level Design

### 6.1 Architecture Diagram

![Facebook News Feed Architecture](../Images/FacebookNewsFeed.excalidraw.svg)

### 6.2 Component Breakdown

The architecture is logically divided into client, edge, service, asynchronous, and storage layers. Each component has a clear responsibility and is independently scalable.

The **Clients** consist of mobile applications on iOS and Android, as well as a responsive web application. These clients handle user authentication, manage local caches and UI state, and construct API requests to interact with the backend. They implement optimistic UI behaviors, such as showing a newly created post immediately in the local feed before receiving confirmation from the server, to improve perceived responsiveness.

The **API Gateway** acts as the unified entry point for all external traffic. It is responsible for authenticating incoming requests via JWT or similar tokens, applying global and user-level rate limits, and routing requests to internal microservices based on HTTP paths and methods. It may also handle cross-cutting concerns such as request/response transformation, CORS policies, and protocol translation from REST/HTTP to internal gRPC calls. The gateway allows backend services to remain relatively simple and focused on business logic while keeping security and traffic management centralized.

The **Feed Service** is responsible for constructing and serving the user news feed. It orchestrates reads from the Precomputed Feed Table, Post cache, Post Table, and Follow Service. It encapsulates the hybrid push-pull logic that combines precomputed items and on-demand celebrity posts. Being stateless and idempotent, it scales horizontally under load by simply adding more instances behind the load balancer.

The **Post Service** manages post creation and retrieval. On writes, it validates content, generates IDs, persists to DynamoDB, and publishes to the Fan-Out Queue for async processing. On reads, it may provide direct post lookups for detail views beyond the feed context. This service is also stateless and horizontally scalable.

The **Follow Service** encapsulates management of follow relationships. It handles the creation and removal of entries in the Follow Table and exposes APIs to list who a user follows and who follows a given user. Because these operations are read-heavy, the service leverages caching aggressively, particularly for frequently accessed follow lists.

The **Async Processing Layer** consists mainly of the **Fan-Out Queue** and **Async Workers**. The queue, using systems such as SQS or Kafka, provides a durable and scalable buffer for post events. The workers consume these events and perform the expensive fan-out process, querying followers and writing entries to the Precomputed Feed Table. The number of worker instances can be scaled up or down dynamically based on queue depth, allowing the system to adapt to traffic spikes such as major global news events.

The **Storage Layer** includes DynamoDB as the primary data store for posts, follows, and precomputed feeds, Redis for caching, and potentially additional replicated caches for hot key mitigation. DynamoDB is chosen for its ability to handle very high write and read throughput while providing low-latency key-value and range queries. It eliminates the need for manual sharding logic typically required with relational databases and offers built-in partitioning and auto-scaling features.

Load balancing across service instances and cache clusters is typically done with a combination of round-robin and least-connections strategies. Stateless HTTP services such as the Feed, Post, and Follow Services can use round-robin distribution. Long-lived connections such as websockets for future real-time features can benefit from least-connections algorithms to avoid overload on a subset of instances.

---

## 7. Deep Dives

### 7.1 Fan-Out Problem: Handling Users with Many Followers

The core challenge of a news feed is deciding how to deliver new posts to followers at scale. The naive push model, where every new post is immediately written into the feed of every follower, quickly becomes intractable for users with millions of followers. A single post from such a user could trigger tens or hundreds of millions of writes, creating extreme write amplification and uneven load across worker processes and database partitions.

A pure pull model, however, where each user's feed is generated on demand by querying recent posts from every user they follow, solves the write amplification but creates expensive and slow read operations. If a user follows a few thousand accounts, a naive implementation would need to issue thousands of queries or a large aggregated query, leading to high latency and heavy database load at read time.

The system adopts a **hybrid push-pull model**, which combines the best of both worlds. For the vast majority of users with modest follower counts, it uses a push model: when they create a post, the system precomputes and stores feed items for each follower. These followers then enjoy very fast reads from the Precomputed Feed Table. For users who exceed a certain follower-count threshold, typically considered celebrities, the system switches to a pull model. Their posts are not fanned out to each follower's feed; instead, followers pull these posts on-demand when they request their feed.

This hybrid approach requires logic in the Async Workers that process the Fan-Out Queue. When a new post event arrives, the worker first checks the follower count of the author. If this count is below a defined `CELEBRITY_THRESHOLD` (e.g., 100,000), it proceeds to retrieve followers and writes entries into the Precomputed Feed Table for active followers. If the follower count exceeds the threshold, the worker marks the post as non-precomputed, effectively registering it as a celebrity post that will be handled via pull at read time.

At feed retrieval time, the Feed Service queries the Precomputed Feed Table to obtain the majority of feed items and then supplements these with on-demand posts from celebrity users. It does this by identifying which followed users are marked as celebrities and issuing limited, recent-post queries against the Post Table for each such user. These posts are then merged with the precomputed items and sorted chronologically. With this design, the heavy, unbounded write work for celebrity posts is avoided, while ordinary users still enjoy low-latency, precomputed feeds.

### 7.2 Hot Key Problem: Handling Viral Posts

Another central challenge is the **hot key** problem. When a post goes viral, millions of users may request it in a short time window. If all of these reads are directed at a single partition key in DynamoDB, the underlying partition can become saturated, triggering throttling and degraded performance.

Conventional wisdom suggests sharding keys (for instance, adding random suffixes to keys) or using a sharded cache keyed by postId. However, sharding a cache by postId only ensures that a particular post is handled by one cache shard; for a viral post, all traffic to that post still converges on that single shard, creating another hotspot in the cache layer rather than the database.

The design here uses a **replicated cache** strategy, where multiple independent cache clusters (for example, ten Redis clusters) can each contain a copy of the same post. Application instances select an arbitrary cache instance (often randomly) for each request. If the post is already cached there, the request is served immediately. If not, the service fetches the post from DynamoDB, writes it into that specific cache instance with a TTL, and returns the content.

This means that for a viral post, its value will be stored in multiple Redis clusters, each serving a fraction of the requests. The cost is duplication of cache entries, but the benefit is distributing load across many cache nodes. In the worst case, when the post is not cached anywhere yet, the system will issue at most one database request per cache instance, instead of millions of direct database reads. This drastically reduces the likelihood of database throttling or latency spikes due to a single hot key.

The trade-off between sharded and replicated caches is between memory efficiency and hot key resistance. A sharded cache is more memory efficient since each key is stored once, but it does not mitigate hotspots. A replicated cache uses more memory for duplicate entries, but it is highly resilient to sudden spikes in access to specific keys. Given the scale of viral content and the relatively low cost of memory compared to the risk of widespread database contention, the replicated cache strategy is adopted as the preferred approach for hot key mitigation.

### 7.3 Database Partitioning and Indexing

Partitioning strategy is central for achieving high throughput and avoiding hotspots in DynamoDB. The Post Table uses a `postId`-based partition key to achieve even data distribution. `postId` values are typically random UUIDs, which naturally distribute writes across many partitions. To support the common query pattern of fetching recent posts by a particular user, a Global Secondary Index keyed by `userId` with `createdAt` as the sort key is added. This allows efficient retrieval of posts for a given user in chronological order without scanning unrelated data.

The Follow Table's primary key is `userFollowing`, representing the user who follows others, with `userFollowed` as the sort key. This makes it efficient to retrieve all users a given user follows. To retrieve followers of a user, the system uses a GSI where the partition key is `userFollowed` and sort key is `userFollowing`. These design choices align with the most common access patterns: "who do I follow?" and "who follows me?"

For the Precomputed Feed Table, using `userId` as the partition key and `timestamp#postId` as the sort key provides natural sharding across users and automatic chronological ordering within a user's feed. The range queries by sort key allow pagination and incremental retrieval of feed items. To keep partition sizes bounded, only the most recent N feed items per user are stored, with older items eligible for truncation or on-demand fetching.

Index design must account for cost and performance. Sparse GSIs are used where appropriate, and attributes indexed in GSIs are chosen carefully to match query patterns. Composite sort keys embed multiple pieces of ordering information, such as the timestamp and post ID, enabling complex queries and stable cursors without extra round trips. The overall schema and indexing strategy are tailored to match the limited but high-volume access patterns typical of a news feed.

### 7.4 Caching Strategy

The caching strategy is multi-layered, combining local in-process caches with centralized distributed caches and replicated caches for hot content. A small **L1 cache** in each service instance stores recently accessed items (for example, user profiles, follow lists, or feed fragments) for milliseconds to seconds. This reduces repeated calls to Redis and DynamoDB and is particularly effective for repeated accesses within a single user session.

A **Redis L2 cache** acts as the primary shared cache across service instances. It stores hot posts, user data, and follow lists with moderate time-to-live (TTL) values. For example, post content might have a TTL of one hour, while follow lists and user profiles have TTLs of one day. Redis cluster sizes and memory budgets are set to meet hit rate and latency targets while minimizing eviction of frequently accessed data. The system follows a cache-aside pattern: on a cache miss, it fetches from DynamoDB, then populates Redis before returning the result.

The **Replicated Cache** layer is specifically for handling viral posts and other hot keys. Multiple Redis clusters exist in parallel, each independent of the others. Application nodes pick a cache instance deterministically or randomly on each read. When a post becomes hot, it is cached redundantly across many clusters, ensuring that requests are balanced across hardware and do not overload any single node or partition.

Cache invalidation complexity is reduced by making posts immutable. Users cannot edit posts; they can only create or delete them, which simplifies cache management because a cached post either exists or not. For mutable data such as follow relationships, the system uses explicit invalidation: when a follow or unfollow operation occurs, the corresponding cached follow list entry is deleted. On the next read, the data is fetched from DynamoDB and re-cached. TTLs also serve as a natural backstop, ensuring that any stale data is eventually refreshed even if invalidation messages are missed.

To protect against cache penetration, where a large number of requests target keys that do not exist in the database, the system may employ a Bloom filter to quickly determine if a given `postId` or `userId` could exist. If the Bloom filter indicates non-membership, the request can be rejected early without hitting the database. Additionally, negative results (for example, "post not found") can be cached for a short time to avoid repeated database lookups for invalid IDs.

### 7.5 Scaling Read and Write Paths

Scaling the read path to handle roughly 463,000 QPS (and up to 1.4 million QPS at peak) requires a combination of denormalization, caching, and horizontal scaling. Precomputing feeds transforms what would be many expensive queries into a single key-value lookup per user, providing O(1) access to a user's feed entries. Redis and in-process caches serve a large portion of requests without touching the database, aiming for cache hit rates above 80%. Horizontally scaling the Feed Service instances, combined with load balancing, ensures that CPU and network load are distributed.

The write path must sustain around 46,000 posts per second on average, with peaks much higher. DynamoDB's scalable write capacity, combined with good partition key design, ensures that the Post Table can absorb this volume. The Fan-Out Queue decouples write spikes from worker capacity; even if workers temporarily fall behind, messages remain durable in the queue and are processed when capacity allows. Workers are configured to batch writes to DynamoDB where possible (for example, using batch write APIs to write multiple feed entries per call), increasing throughput and reducing per-item overhead.

Selective fan-out significantly reduces write amplification. By skipping precomputation for celebrity users and possibly deprioritizing inactive users, the system avoids doing an enormous amount of work for users who either do not benefit from precomputation or engage less frequently. Autoscaling within the worker fleet, triggered by queue depth and worker CPU utilization, ensures that more compute is applied during surges, such as global events that cause bursts of posting and engagement.

Overall, the combination of precomputation, caching, asynchronous processing, and horizontal scaling in both services and storage layers ensures that both read and write paths can meet aggressive throughput and latency requirements.

### 7.6 Failure Modes and Resilience

Designing for resilience means anticipating how various components may fail and how the system should behave under such conditions. One common failure mode is **DynamoDB partition throttling**, where a specific partition experiences more read or write throughput than its configured capacity. When this occurs, the system receives throttling exceptions. Clients, such as the Feed or Post Services, implement exponential backoff with jitter to avoid synchronized retry storms. The replicated cache strategy additionally reduces the load on DynamoDB in hot key scenarios, making partition throttling less likely for viral content. During transient issues, the system may continue serving stale data from cache while logging errors and emitting alerts.

Another failure mode is **cache cluster outage**. If a Redis cluster becomes unavailable due to network issues or crashes, the services using that cache will see failed connections or timeouts. To prevent cascading failures, a circuit breaker pattern is used: once a certain number of cache errors occur, the service temporarily stops attempting cache operations and serves data directly from DynamoDB. This leads to degraded but still functional performance. When the cache cluster recovers, the circuit breaker can close and caching resumes.

If **Async Workers** fall behind, indicated by a rapidly increasing queue depth or message age, feed precomputation becomes delayed. Users may see stale feeds or delayed appearance of new posts. Autoscaling workers based on queue metrics can alleviate temporary overload. In extreme situations, lower-priority fan-out work—for example, precomputing feeds for long-inactive users—can be dropped or delayed further to ensure active users receive higher priority. Falling back to the hybrid pull model means that even if precomputation is delayed, followers can still see posts by querying authors' recent posts on demand.

If the **Feed Service** itself becomes overloaded due to a surge in traffic, auto-scaling and rate limiting protect the rest of the system. Rate limiting ensures that abusive or excessive clients do not exhaust resources for everyone else. In worst-case scenarios, the system can respond with HTTP 503 errors for some requests and instruct clients to back off and retry later. Careful use of caching and smaller page sizes can help preserve acceptable latency for the majority of users.

Each of these failure modes is tied into an observability strategy. Metrics such as error rates, queue depth, cache hit ratio, and DynamoDB throttling events trigger alerts via systems like PagerDuty. Runbooks provide engineers with predefined mitigation steps, such as manually scaling workers or adjusting rate limits temporarily.

### 7.7 Security and Privacy

Security and privacy considerations are woven through every layer of the architecture. Authentication is handled via short-lived JWTs issued after a successful login. Tokens include claims such as the `userId`, roles, and expiration time. The API Gateway validates tokens on each request, ensuring that only authenticated calls reach the Feed, Post, or Follow Services. HTTPS is enforced end-to-end to prevent eavesdropping or tampering.

Rate limiting is critical for both availability and abuse prevention. Token bucket or leaky bucket algorithms are implemented, often using Redis to store per-user and per-IP counters. These guard against brute-force attempts, scraping, or distributed denial-of-service attacks. Limits can be tuned by endpoint type; for example, feed reads may allow more requests per minute than post creations.

Data sanitization is also essential. All user-supplied content is validated for size and filtered to remove or escape HTML tags, scripts, and potentially dangerous markup. Even if content is stored as-is, rendering layers in the client ensure that scripts are never executed and markup is safely escaped, preventing cross-site scripting (XSS) attacks. Since the system uses NoSQL APIs with structured programmatic access instead of raw queries, traditional SQL injection attacks are not applicable, but robust validation still mitigates other injection vectors.

Privacy considerations include ensuring that users only see posts they are authorized to see. While the initial design assumes public posts, future extensions could introduce private or friends-only posts, which would require more complex authorization checks when constructing feeds and precomputing feed entries. Audit logs record access to sensitive operations, such as changes in follow relationships, and can aid in forensic analysis and compliance.

### 7.8 Monitoring and Observability

Robust observability enables rapid detection, diagnosis, and resolution of issues. Key metrics include end-to-end latency for feed reads and post writes, broken down by percentile; error rates by endpoint; cache hit and miss rates; DynamoDB read and write latencies; and the depth and processing lag of the Fan-Out Queue. Thresholds and SLOs are defined, such as maintaining p99 feed read latency under 500 ms, cache hit rates above 80%, and queue lag under one minute for normal operation.

Logging is structured, using JSON logs that include correlation IDs, user IDs (where appropriate), timestamps, and contextual fields such as endpoint name and error codes. Logs are aggregated and indexed using tools such as Elasticsearch and viewed via dashboards like Kibana or Grafana. Logs are retained in hot storage for fast search over a 30-day window and archived in cold storage such as S3 for regulatory or investigative purposes over longer periods.

Distributed tracing complements logs and metrics by providing an end-to-end view of a single request as it traverses multiple services and external dependencies. Using systems like AWS X-Ray or Jaeger, each request is assigned a trace ID, and spans are recorded for calls to DynamoDB, Redis, the Fan-Out Queue, and other services. Sampling allows capturing a representative subset of requests, with biased sampling for error cases to aid debugging.

Alerting policies define what conditions trigger pages to on-call engineers. For example, a sustained error rate of more than 1% across key endpoints, or DynamoDB throttling at sustained levels, might raise a P1 or P0 incident. Dashboards provide overviews of system health so that engineers can quickly see whether an incident is localized or systemic.

---

## 8. References

Visual:  
Facebook News Feed Architecture Diagram: `../Images/FacebookNewsFeed.excalidraw.jpg`

Video:  
Design FB News Feed System Design Interview:  
https://www.youtube.com/watch?v=Qj4-GruzyDU

Deep Dive (Primary Specification):  
Hello Interview – Design Facebook's News Feed:  
https://www.hellointerview.com/learn/system-design/problem-breakdowns/fb-news-feed

Additional Contextual Resources (Optional):  
DynamoDB Social Network Schema Design  
News Feed System Design Guides  
Facebook News Feed Algorithm Evolution

---

## 9. Trade-offs and Alternatives

### 9.1 Database Choice

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| DynamoDB | Fully managed, auto-scaling, high write throughput, natural key-value/time-series fit | Vendor lock-in, query flexibility constraints | Chosen as primary store |
| Cassandra | Open source, linearly scalable, multi-DC replication | Operational complexity, tuning and maintenance overhead | Viable alternative for self-managed deployment |
| MySQL / RDBMS | Strong consistency, rich querying and joins | Complex manual sharding at this scale, write bottlenecks, expensive distributed joins | Not suitable for global-scale news feed writes |
| MongoDB | Flexible document model, rich querying | Less optimal for high-write, hot-partition workloads; scaling and consistency trade-offs | Not preferred for this workload |

The decision to use DynamoDB is driven by its managed nature and stability under extreme write loads. For organizations preferring self-managed infrastructure, Cassandra or similar wide-column stores provide an alternative with similar operational patterns but require a mature SRE team.

### 9.2 Fan-Out Strategy

| Strategy | Write Amplification | Read Latency | Operational Complexity | Decision |
|----------|---------------------|--------------|------------------------|----------|
| Pure Push | Very High (N followers per post) | Very Low | High (write spikes, worker scaling) | Not scalable for celebrities |
| Pure Pull | None | High | Moderate (complex read logic, high DB load) | Poor user experience |
| Hybrid Push-Pull | Moderate (selective precomputation) | Low | Higher (mixed logic, queue and workers) | Chosen strategy |

The hybrid approach allows the system to optimize for the common case (users with moderate follower counts) while remaining practical for extreme cases (celebrities, viral events).

### 9.3 Cache Architecture

| Strategy | Hot Key Resilience | Memory Efficiency | Complexity | Decision |
|----------|--------------------|-------------------|------------|----------|
| Sharded Cache by Key | Low (hot key overloads one shard) | High | Low to moderate | Not sufficient for viral posts |
| Replicated Cache | High (load spread across many instances) | Lower (duplicate entries) | Moderate | Chosen for hot keys |
| No Cache | N/A | N/A | Low | Not viable at significant scale |

Given the high ratio of reads to writes and the inevitability of hot content, the replicated cache design is chosen as the most robust way to maintain performance under bursty workloads.

---

## 10. Future Enhancements

Several enhancements can be built atop this foundational design.

Personalized ranking can replace or augment the purely chronological ordering. Machine learning models could score candidate posts based on factors such as user affinity, engagement likelihood, and freshness. The Feed Service would then sort posts by score instead of timestamp alone, possibly still subject to recency constraints. This would increase complexity in feed computation but potentially improve engagement metrics.

Real-time updates via mechanisms like WebSockets or server-sent events would allow users to see new posts appear instantly without pulling to refresh. This would introduce new infrastructure primitives, such as connection managers and pub/sub channels mapped to users or feeds, and would require carefully balancing live updates with rate limits and backpressure.

Support for rich media, such as videos and images, would introduce pipelines for media upload, transcoding, storage (typically S3 or similar object storage), and content delivery via CDNs. Feed entries would then reference media URLs rather than containing all content inline.

Likes, comments, and reactions would require additional tables or collections for interactions, as well as counter caches for aggregate metrics like like counts or comment counts. Efficiently updating and serving these counters at high volume is its own design problem, likely involving incremental aggregation and read-optimized denormalized fields.

Privacy settings and access control lists would add another dimension to feed construction. Posts could be classified as public, friends-only, or custom-visibility, and the Feed and Precomputed Feed pipelines would need to respect these constraints, performing visibility checks prior to inserting entries into feeds or returning them in API responses.

Lastly, multi-region active-active deployments could be introduced to reduce latency for global users and improve resilience to regional outages. This would bring cross-region replication challenges, conflict resolution for mutable entities like follow relationships, and potentially per-region feed materialization strategies that reflect the global state.

---

**Document Version:** 1.0  
**Last Updated:** December 2, 2025  
**Author:** Principal Software Architect  
**Status:** Production Ready
