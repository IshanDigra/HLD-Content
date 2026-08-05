# Facebook Post Search System Design

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
- [ ] Users can search for posts they have written or posts written by their friends.
- [ ] Search results should be sorted by relevance or time.
- [ ] Search results must respect privacy settings (only show posts the searcher is allowed to see).

### Non-Functional Requirements
- [ ] **High Scalability**: Billions of posts and millions of searches per second.
- [ ] **Low Latency**: Search results should return in < 200ms.
- [ ] **Real-time Indexing**: New posts should be searchable almost immediately.

---

## 2. Core Entities (3-5 min)

- **Post**: `postId`, `authorId`, `content`, `timestamp`, `privacyScope`
- **User**: `userId`, `friendList`

---

## 3. API Design (~5 min)

### `GET /api/v1/search/posts`
- **Parameters**: `query` (e.g., "birthday party"), `cursor`, `limit`
- **Response**: List of matching `Post` objects.

---

## 4. Data Flow (5-10 min)

1. User creates a post -> Post is saved in DB -> A Kafka event is fired -> Indexer consumes event and updates the Search Index.
2. User searches -> API Gateway -> Search Service -> Queries Index for matching terms -> Filters results by user's social graph (friends) -> Ranks and returns.

---

## 5. High-Level Design (15-20 min)

- **Search Index (Inverted Index)**: Maps terms to `postIds`.
- **Social Graph DB**: Stores relationships (who is friends with whom).
- **Search Aggregator**: Scatters the search query across multiple index shards, gathers the results, filters them based on permissions, and sorts them.
- **Cache**: Caches the social graph and popular search queries.

---

## 6. Deep Dives (15-20 min)

### Graph Intersection (The Core Challenge)
- **Challenge**: A simple full-text search for "birthday" will return millions of posts. Filtering them *after* retrieval to check if the author is a friend is too slow.
- **Solution (Term-at-a-time vs. Document-at-a-time)**:
  - We must intersect the Inverted Index (posts containing "birthday") with the Social Graph (posts authored by friends).
  - To do this efficiently, the Search Index can be partitioned by `authorId` instead of `documentId`.
  - The Search Aggregator queries the Social Graph to get the list of the user's friends. It then queries the specific Index Shards holding the posts for those friends, asking for posts containing "birthday".

---

## 7. Address Key Issues (5 min)

### Index Partitioning
- Partitioning by `Term` vs `Document` vs `Author`. For social search, partitioning by `Author` (or a hash of `AuthorId`) ensures that all posts by a specific user are on the same machine, optimizing the social graph intersection.
