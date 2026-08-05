# Load Balancer System Design

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
- [ ] System must distribute incoming network traffic across multiple healthy servers.
- [ ] System should support Layer 4 (TCP/UDP) and Layer 7 (HTTP/HTTPS) load balancing.
- [ ] System must perform continuous Health Checks to ensure traffic is only routed to active servers.

### Non-Functional Requirements
- [ ] **High Scalability**: Must handle up to 1 million requests per second (peak traffic).
- [ ] **High Availability**: The Load Balancer itself must not be a single point of failure (SPOF).
- [ ] **Low Latency**: Traffic routing should add negligible overhead.

---

## 2. Core Entities (3-5 min)

- **Target Server**: `ip_address`, `port`, `weight`, `health_status`
- **Listener**: `port`, `protocol` (TCP, HTTP)
- **Routing Rule**: `url_path`, `target_group`
- **Health Check Config**: `interval`, `timeout`, `healthy_threshold`, `unhealthy_threshold`

---

## 3. API Design (~5 min)

*(While a LB intercepts standard web traffic, it requires an API for control/management)*

### `POST /api/v1/targets`
- **Purpose**: Register a new target server.
- **Request Body**: `{ "ip": "10.0.0.5", "port": 8080, "weight": 1 }`
- **Response**: `200 OK`

### `GET /api/v1/targets/health`
- **Purpose**: Get current status of all servers.
- **Response**: `{ "healthy": ["10.0.0.5"], "unhealthy": [] }`

---

## 4. Data Flow (5-10 min)

1. Client sends a request (e.g., HTTP `GET /api/data`).
2. DNS resolves the domain to the IP of the Load Balancer.
3. Request hits the LB. LB terminates the connection (if Layer 7) or inspects packets (if Layer 4).
4. LB determines the appropriate target group based on Rules (e.g., path `/api/data` goes to API servers).
5. LB uses a routing algorithm (e.g., Round Robin) to select a healthy server.
6. LB forwards the request to the server.
7. Server responds to LB, LB forwards response back to Client.

---

## 5. High-Level Design (15-20 min)

- **DNS Layer**: Uses DNS Load Balancing to route users to multiple physical Load Balancer IPs.
- **Load Balancer Nodes (Active-Active)**:
  - Multiple LB nodes working together. If one fails, others take over.
  - Keeps state synchronized (e.g., session stickiness).
- **Control Plane**: Manages configuration, registers servers, applies rules.
- **Data Plane**: The highly optimized networking stack that actually routes the bytes based on rules provided by the Control Plane.
- **Health Checker Component**: A background daemon pinging servers (TCP ping or HTTP GET) every few seconds.

---

## 6. Deep Dives (15-20 min)

### Layer 4 vs Layer 7 Load Balancing
- **Layer 4 (Transport)**: Operates on IP and Port. It does not inspect the HTTP body. Very fast, uses NAT (Network Address Translation). Good for massive scale raw TCP throughput.
- **Layer 7 (Application)**: Operates on HTTP/HTTPS. It terminates the SSL connection, reads headers, cookies, and URL paths. Can make smart routing decisions (e.g., routing `/images/*` to an image server group). Takes more CPU overhead.

### Load Balancing Algorithms
1. **Round Robin**: Distributes requests sequentially. Best if all servers have equal capacity.
2. **Weighted Round Robin**: Servers with higher capacity get a higher weight (more requests).
3. **Least Connections**: Routes to the server with the fewest active connections. Best for long-lived connections (e.g., WebSockets).
4. **IP Hash**: Hashes the client's IP to assign a server. Guarantees the same user always hits the same server (Sticky Sessions).

---

## 7. Address Key Issues (5 min)

### Eliminating the LB as a SPOF
- Use multiple LB instances managed by **VRRP (Virtual Router Redundancy Protocol)** or Keepalived.
- They share a Virtual IP (VIP). If the Master goes down, the Backup instantly assumes the VIP and traffic flows uninterrupted.

### SSL Termination vs SSL Passthrough
- **SSL Termination**: LB decrypts HTTPS traffic, inspects it, and forwards plain HTTP to backend servers. Saves backend CPU, but data in the internal network is unencrypted.
- **SSL Passthrough**: LB forwards encrypted bytes to the backend. Backend does the decryption. More secure, but LB cannot do Layer 7 routing.
