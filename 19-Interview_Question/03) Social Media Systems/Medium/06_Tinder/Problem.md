# Tinder System Design



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

## Table of Contents
![1. Requirements Architecture](../../../../19-interview-questions/Images/1. Requirements.excalidraw.svg)
![2. Core Entities Architecture](../../../../19-interview-questions/Images/2. Core Entities.excalidraw.svg)
![3. API Design Architecture](../../../../19-interview-questions/Images/3. API Design.excalidraw.svg)
![4. Data Flow Architecture](../../../../19-interview-questions/Images/4. Data Flow.excalidraw.svg)
![5. High-Level Design Architecture](../../../../19-interview-questions/Images/5. High-Level Design.excalidraw.svg)
![6. Deep Dives Architecture](../../../../19-interview-questions/Images/6. Deep Dives.excalidraw.svg)
![7. Address Key Issues Architecture](../../../../19-interview-questions/Images/7. Address Key Issues.excalidraw.svg)
![References & Original Diagrams Architecture](../../../../19-interview-questions/Images/References & Original Diagrams.excalidraw.svg)

---
## 1. Requirements (5-10 min)

### Functional Requirements
- [ ] Users can create a profile with photos and bio.
- [ ] Users can view recommendations based on location, age, and gender preferences.
- [ ] Users can swipe right (like) or left (pass).
- [ ] If both users swipe right, a "Match" is created.
- [ ] Matched users can chat.




### Back-of-the-Envelope (BOE) Calculations
**Step 1: Assumptions**
- Users: 50M DAU
- Activity: 100 swipes/user/day
- Payload: Swipe event ~100 bytes. Profile load includes multiple images (few MBs).

**Step 2: Load (QPS)**
- Swipe Write QPS: (50M * 100) / 100,000 ≈ 50,000 QPS

**Step 3: Storage (5-year plan)**
- Heavy image storage (S3) for profiles.

**Step 4: Bandwidth**
- High ingress/egress due to image loading.

### Non-Functional Requirements
- [ ] **Low Latency**: Recommendations must load fast.
- [ ] **Scalability**: High read (viewing profiles) and write (swiping) throughput.
- [ ] **Geospatial**: Fast location-based querying is essential.

---

## 2. Core Entities (3-5 min)

- **User**: `userId`, `location`, `preferences`, `images`
- **Swipe**: `swiperId`, `swipeeId`, `action` (LIKE, PASS), `timestamp`
- **Match**: `matchId`, `user1Id`, `user2Id`, `timestamp`

---

## 3. API Design (~5 min)

### `GET /api/v1/recommendations`
- **Response**: Array of User profiles.

### `POST /api/v1/swipes`
- **Request**: `{ "swipeeId": "456", "action": "LIKE" }`
- **Response**: `{ "matched": true, "matchId": "..." }` (if mutual).

---

## 4. Data Flow (5-10 min)

1. Client requests recommendations -> Gateway -> Recommendation Engine queries Geo-index -> returns profiles.
2. User swipes -> Gateway -> Swipe Service logs the swipe.
3. If LIKE, checks if `swipee` already liked `swiper` in cache/DB. If yes, creates Match and notifies both via Push Notification.

---

## 5. High-Level Design (15-20 min)

### High-Level Architecture
```mermaid
graph TD
    A[Load Balancer] --> B[Service Cluster]
    B --> C[(Primary DB)]
    C -.->|Async Replication| D[(Read Replica)]
    B --> E[(Redis Cache)]
```




- **Profile Service**: Manages user data. Images in S3.
- **Recommendation Engine**: Pre-computes and caches batches of profiles for users.
- **Geospatial DB**: Elasticsearch or Redis Geo to query users within a radius.
- **Swipe Service**: Handles the high-throughput swipe events. Uses message queues (Kafka) to process matches asynchronously.
- **Chat Service**: Similar to WhatsApp/Slack for matched users.

---

## 6. Deep Dives (15-20 min)

### Deep Dive / Data Flow
```mermaid
sequenceDiagram
    participant Client
    participant API_Gateway
    participant Service
    participant Database

    Client->>API_Gateway: Request
    API_Gateway->>Service: Route
    Service->>Database: Query/Update
    Database-->>Service: Result
    Service-->>API_Gateway: Response
    API_Gateway-->>Client: Result
```






### Geospatial Indexing & Recommendation Batching
- **Challenge**: Querying for users within 10 miles constantly is expensive.
- **Solution**: Use Geohashing or Quadtrees. The Recommendation Engine runs a background worker that pre-fetches a queue of 20-50 profiles for a user and stores them in Redis. When the user opens the app, they instantly get the pre-calculated queue.

### Swipe & Match Logic
- **Challenge**: Checking for mutual likes synchronously on every right swipe slows down the API.
- **Solution**:
  - When User A likes User B, store `(A, B) -> LIKE` in a fast NoSQL DB (e.g., Cassandra).
  - Also check if `(B, A) -> LIKE` exists. To make this fast, keep recent likes in a Redis Cache.
  - If a match occurs, publish a `MatchEvent` to Kafka, which updates the Match DB and sends push notifications out of the critical path.

---

## 7. Address Key Issues (5 min)

### Fault Tolerance & Resiliency
- Caching layer failure: Fall back to DB, but expect degraded performance. Geohash DB (Elasticsearch) must be highly replicated.

## References & Original Diagrams
![Tinder Architecture](../../../../19-interview-questions/Images/Tinder.excalidraw.svg)
