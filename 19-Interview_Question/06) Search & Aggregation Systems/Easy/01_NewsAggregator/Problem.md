# News Aggregator System Design

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
- [ ] System automatically crawls and extracts news from various RSS feeds or websites.
- [ ] Users can view a chronological or personalized feed of aggregated news.
- [ ] Users can search for news articles by keywords.

### Non-Functional Requirements
- [ ] **High Availability**: The news feed reading side must be highly available.
- [ ] **Scalability**: Must handle high read traffic and a continuously growing dataset of articles.
- [ ] **Low Latency**: Search and feed generation must be fast.

---

## 2. Core Entities (3-5 min)

- **Source**: `sourceId`, `url`, `type` (RSS, Web), `crawlFrequency`
- **Article**: `articleId`, `sourceId`, `title`, `content`, `url`, `publishedAt`
- **User**: `userId`, `preferences`

---

## 3. API Design (~5 min)

### `GET /api/v1/news/feed`
- **Purpose**: Get aggregated news feed for the user.
- **Parameters**: `cursor`, `limit`
- **Response**: List of `Article` objects.

### `GET /api/v1/news/search`
- **Purpose**: Search articles.
- **Parameters**: `q` (query string).

---

## 4. Data Flow (5-10 min)

1. **Ingestion**: Scheduler triggers crawler to fetch RSS feeds -> HTML/XML parsed -> Articles saved to DB and indexed in ElasticSearch.
2. **Serving**: User requests feed -> API queries Cache or DB -> Returns articles.
3. **Searching**: User searches -> API queries ElasticSearch -> Returns matching articles.

---

## 5. High-Level Design (15-20 min)

- **Feed Aggregator/Crawler**: Background workers (e.g., Celery/Kafka consumers) that fetch and parse external feeds.
- **Deduplication Service**: Checks if the article was already fetched (using URL hash or content hashing).
- **Relational DB / NoSQL**: Stores the raw article text and metadata (Cassandra for high volume, PostgreSQL if smaller).
- **Search Engine**: ElasticSearch cluster to index article content and enable fast full-text searches.
- **Cache**: Redis for caching the top news feed and recent queries.

---

## 6. Deep Dives (15-20 min)

### Content Deduplication
- **Challenge**: Multiple news sites might report the exact same AP/Reuters wire story. Showing the same story 5 times ruins the UX.
- **Solution**: Use Simhash or MinHash algorithms. Unlike MD5 (which changes entirely if a single comma is added), Simhash generates similar hashes for similar text. We can group articles with high similarity scores and only show the canonical version to the user.

---

## 7. Address Key Issues (5 min)

### Fault Tolerance & Resiliency
- Ensure crawlers respect `robots.txt` and employ exponential backoff if a target news site goes down, to avoid infinite failing loops.
