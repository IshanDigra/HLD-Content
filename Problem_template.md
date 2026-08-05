# [Problem Name]

| Step | Focus Area | Time Allocation | Key Activities |
|------|-----------|----------------|----------------|
| 1 | Requirements | 5-10 min | Clarify functional and non-functional requirements, identify core features |
| 2 | Core Entities | 3-5 min | Identify main entities and their relationships |
| 3 | API Design | ~5 min | Define key endpoints, methods, and data structures (keep brief) |
| 4 | Data Flow (Optional) | 5-10 min | Map request/response flows, identify data movement patterns |
| 5 | High-Level Design | 15-20 min | Draw system architecture, components, and their interactions |
| 6 | Deep Dives | 15-20 min | Dive into critical components, address bottlenecks and optimizations |
| 7 | Address Key Issues | 5 min | Handle failures, security, monitoring, and edge cases |

---

## 1. Requirements (5-10 min)

### Functional Requirements
- [ ] Feature 1 (e.g., User can upload a video)
- [ ] Feature 2 (e.g., User can view a video)

### Non-Functional Requirements
- [ ] Availability vs Consistency (CAP Theorem considerations)
- [ ] Latency requirements (e.g., 200ms for read requests)
- [ ] Scalability (Traffic, Data Storage, Bandwidth estimates)

---

## 2. Core Entities (3-5 min)

- **Entity 1 (e.g., User)**: Attributes (id, name, email)
- **Entity 2 (e.g., Video)**: Attributes (id, metadata, url)
- **Relationships**: Entity 1 has many Entity 2, etc.

---

## 3. API Design (~5 min)

### `POST /api/v1/resource`
- **Purpose**: Create a new resource.
- **Request Body**:
  ```json
  {
    "field1": "value1"
  }
  ```
- **Response**: `201 Created`

### `GET /api/v1/resource/:id`
- **Purpose**: Retrieve a resource.
- **Response**: `200 OK`

---

## 4. Data Flow (5-10 min) *(Optional for Data Intensive Designs)*

1. Client sends a request to the Load Balancer.
2. Load Balancer routes to the appropriate Service.
3. Service queries the Database/Cache.
4. Response is sent back to the Client.

---

## 5. High-Level Design (15-20 min)

*(Describe the core components of the system here. Mention Load Balancers, API Gateways, Microservices, Databases, Caches, Message Queues, etc.)*

- **Component 1 (e.g., API Gateway)**: Handles rate limiting and routing.
- **Component 2 (e.g., Metadata DB)**: Stores relational data.
- **Component 3 (e.g., Object Storage)**: Stores media files.

---

## 6. Deep Dives (15-20 min)

### Key Component / Bottleneck
- **Challenge**: E.g., Database write contention.
- **Solution**: E.g., Introduce a Message Queue (Kafka) to decouple writes, or Database Sharding.
- **Trade-offs**: E.g., Eventual consistency vs strict consistency.

---

## 7. Address Key Issues (5 min)

### Fault Tolerance & Resiliency
- **Single Points of Failure (SPOFs)**: E.g., Use active-passive or active-active setups.
- **Replication**: E.g., DB replication for read-heavy workloads.

### Security
- E.g., Authentication (JWT), Rate Limiting.

### Monitoring & Observability
- E.g., Track latency, error rates, and system throughput using Prometheus/Grafana.
