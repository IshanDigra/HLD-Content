# Ad Click Aggregator System Design

| Step | Focus Area | Time Allocation | Key Activities |
|------|-----------|----------------|----------------|
| 1 | Requirements | 5-10 min | Clarify functional and non-functional requirements, identify core features |
| 2 | Core Entities | 3-5 min | Identify main entities and their relationships |
| 3 | API Design | ~5 min | Define key endpoints, methods, and data structures (keep brief) |
| 4 | Data Flow | 5-10 min | Map request/response flows, identify data movement patterns |
| 5 | High-Level Design | 15-20 min | Draw system architecture, components, and their interactions |
| 6 | Deep Dives | 15-20 min | Dive into critical components, address bottlenecks and optimizations |
| 7 | Address Key Issues | 5 min | Handle failures, security, monitoring, and edge cases |

---

## 1. Requirements (5-10 min)

### Functional Requirements
- [ ] Aggregate ad clicks over various time windows (last 1 minute, 1 hour, 1 day).
- [ ] Support queries to retrieve aggregated click counts for a specific Ad ID.
- [ ] Provide reporting dashboards for advertisers.

### Non-Functional Requirements
- [ ] **High Throughput**: Must handle tens of thousands of click events per second.
- [ ] **Accuracy / Exactly-Once Processing**: Clicks are tied to revenue. We cannot overcount or undercount clicks.
- [ ] **Low Latency Reads**: Dashboards should load quickly.

---

## 2. Core Entities (3-5 min)

- **ClickEvent**: `adId`, `userId`, `timestamp`, `ipAddress`
- **AggregatedMetrics**: `adId`, `timeWindow` (e.g., `2024-05-12T10:00Z`), `clickCount`

---

## 3. API Design (~5 min)

### `POST /api/v1/clicks`
- **Purpose**: Record a click.
- **Request**: `{ "adId": "123", "userId": "abc", "timestamp": "..." }`

### `GET /api/v1/metrics/ads/:id`
- **Purpose**: Get click counts for an ad.
- **Parameters**: `startTime`, `endTime`, `resolution` (minute, hour, day).

---

## 4. Data Flow (5-10 min)

1. User clicks an Ad -> API Gateway -> Click gets appended to a Message Queue (Kafka).
2. Stream Processing Engine (Flink/Spark) consumes the stream, deduplicates clicks, and aggregates counts using Tumbling/Sliding windows.
3. Aggregated counts are written to an OLAP Database or Time-Series Database (e.g., Apache Druid or Cassandra).
4. Advertiser Dashboard queries the Database via the Analytics API.

---

## 5. High-Level Design (15-20 min)

- **Kafka**: Highly durable, partitioned message queue to absorb the massive write load of click events.
- **Apache Flink**: Real-time stream processing engine that supports exactly-once processing semantics and windowing functions.
- **Time-Series DB (Cassandra / Druid)**: Optimized for fast writes and time-range aggregation queries. Storing raw data in a traditional SQL DB would bottleneck quickly.
- **Batch Processing (Hadoop)**: A secondary offline pipeline (Lambda architecture) that re-processes raw logs nightly to ensure 100% accuracy in billing.

---

## 6. Deep Dives (15-20 min)

### Exactly-Once Processing & Deduplication
- **Challenge**: A user clicks twice by mistake, or network lag causes the client to retry sending the click event. We shouldn't charge the advertiser twice.
- **Solution**:
  - Generate a unique `clickId` on the client side when the ad is rendered.
  - The Stream Processor maintains a fast Bloom Filter or Redis cache of recently seen `clickId`s (for the last 10 minutes) to drop duplicates.
  - Flink provides built-in exactly-once state snapshots so if a worker crashes, it doesn't process the same Kafka message twice.

### Dealing with Late Events (Watermarks)
- **Challenge**: A user clicks on their mobile phone while in a tunnel. The event reaches the server 5 minutes late.
- **Solution**: Use event-time processing (not processing-time). Flink uses "Watermarks" to allow a grace period for late-arriving events before finalizing a time window and writing it to the database.

---

## 7. Address Key Issues (5 min)

### Storage Optimization
- Raw click logs are moved from Kafka to cheap S3 storage after a few days. The fast Time-Series DB only holds pre-aggregated data (e.g., `Ad 123 got 50 clicks between 10:00 and 10:01`), saving massive amounts of space.
