# Key Value Store System Design

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
- [ ] Clients can execute `put(key, value)` to store data.
- [ ] Clients can execute `get(key)` to retrieve data.
- [ ] Values are opaque byte arrays.

### Non-Functional Requirements
- [ ] **High Scalability**: Must scale out horizontally to handle petabytes of data.
- [ ] **High Availability**: System must remain available for reads and writes even during node failures.
- [ ] **Low Latency**: Sub-millisecond latency for reads and writes.

---

## 2. Core Entities (3-5 min)

- **Record**: `key` (string/bytes), `value` (bytes), `timestamp`/`version`

---

## 3. API Design (~5 min)

### `put(key, value)`
- Stores the value against the key.

### `get(key)`
- Returns the value associated with the key.

---

## 4. Data Flow (5-10 min)

1. Client sends a request to any Node (Coordinator) in the cluster.
2. Coordinator hashes the key to determine which replica nodes hold the data.
3. Coordinator forwards the request to the replicas.
4. Once a quorum (W or R) responds, the Coordinator returns the result to the client.

---

## 5. High-Level Design (15-20 min)

- **Consistent Hashing Ring**: Distributes data evenly across the cluster.
- **Data Replication**: Data is replicated to `N` consecutive nodes on the hash ring.
- **Gossip Protocol**: Decentralized mechanism for nodes to discover each other and detect failures.
- **SSTable / LSM Trees**: The on-disk storage engine optimized for high write throughput.

---

## 6. Deep Dives (15-20 min)

### CAP Theorem & Quorum Consensus
- **Challenge**: Network partitions happen. We must choose between Consistency and Availability.
- **Solution (Dynamo-style)**: We choose Availability and Eventual Consistency (AP).
  - Define `N` (Replicas), `W` (Write Quorum), `R` (Read Quorum).
  - Rule: `W + R > N` guarantees strong consistency. If we relax this (e.g., `W=1, R=1`), we get high availability and low latency at the cost of eventual consistency.

### Resolving Data Conflicts (Vector Clocks)
- **Challenge**: Since we allow concurrent writes to different replicas during network partitions, data can diverge.
- **Solution**: Use Vector Clocks (a list of `[server, version]` pairs) attached to every value. If a client reads conflicting versions, the client must resolve the conflict and write back the merged result.

---

## 7. Address Key Issues (5 min)

### Fault Tolerance (Hinted Handoff & Merkle Trees)
- **Temporary Failure**: If Node A is down, Node B accepts the write on its behalf (Hinted Handoff). When A returns, B pushes the data to A.
- **Permanent Failure / Anti-entropy**: Nodes compare Merkle Trees (hash trees of their data ranges) periodically in the background to detect and repair missing data efficiently.
