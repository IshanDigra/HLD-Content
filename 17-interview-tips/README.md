# Interview Tips

A comprehensive guide to mastering system design interviews with structured approaches, practical tips, and key concepts for building scalable systems.

---

## Table of Contents

1. [Six-Step Problem Solving Approach](#six-step-problem-solving-approach)
2. [Answering Framework](#answering-framework)
3. [Back of the Envelope Calculation](#back-of-the-envelope-calculation)
4. [API Design](#api-design)
5. [Choosing the Right Database](#choosing-the-right-database)
6. [Covering Non-Functional Requirements](#covering-non-functional-requirements)
7. [Scaling from 0 to 1 Million Users](#scaling-from-0-to-1-million-users)
8. [Diagram Making Tips](#diagram-making-tips)
9. [Miscellaneous Key Points](#miscellaneous-key-points)
10. [Quick Reference Tables](#quick-reference-tables)

---

## Six-Step Problem Solving Approach

This structured approach ensures comprehensive coverage of system design problems. Follow these steps sequentially for optimal time management.

| Step | Focus Area | Time Allocation | Key Activities |
|------|-----------|----------------|----------------|
| 1 | Requirements | 5-10 min | Clarify functional and non-functional requirements, identify core features |
| 2 | Core Entities | 3-5 min | Identify main entities and their relationships |
| 3 | API Design | ~5 min | Define key endpoints, methods, and data structures (keep brief) |
| 4 | Data Flow | 5-10 min | Map request/response flows, identify data movement patterns |
| 5 | High-Level Design | 15-20 min | Draw system architecture, components, and their interactions |
| 6 | Deep Dives | 15-20 min | Dive into critical components, address bottlenecks and optimizations |

**Important Notes:**
- Focus on the most critical 3 features initially
- Keep other features on hold until main design is solid
- Don't spend more than 5 minutes on API Design unless specifically asked
- API Design is often the default approach - save time for deep dives

---

## Answering Framework

### Step-by-Step Interview Approach

| Step | Focus Area | Key Activities |
|------|-----------|----------------|
| 1. Clarify Requirements | Functional & Non-Functional | Ask clarifying questions, identify core features, understand constraints |
| 2. Capacity Estimation | Scale & Volume | Calculate users, traffic, storage, memory, and network requirements |
| 3. High-Level Design | System Architecture | Draw major components, define data flow, identify key services |
| 4. Database Design | Data Modeling & Storage | Choose database type, design schema, define access patterns |
| 5. API Design | Interface Definition | Define endpoints, request/response formats, communication protocols |
| 6. Deep Dive | Component Details | Scale specific components, address bottlenecks, optimize performance |
| 7. Address Key Issues | Trade-offs & Reliability | Handle failures, security, monitoring, and edge cases |

### Critical Interview Tips

**Initial Approach:**
- Treat the interview as a collaborative discussion with a coworker
- Ask clarifying questions early; some interviewers prioritize this heavily
- Focus on 3 core features initially; keep others on hold
- Create an overview picture before diving deep into specific components
- Keep time management in mind throughout

**Key Questions to Ask:**

*Users & Usage:*
- Who are the users?
- How will they use it?
- What are the specific use cases?
- What are the inputs and outputs?

*Scale & Performance:*
- How much data do we expect to handle?
- How many requests per second do we expect?
- What is the read-to-write ratio?
- What are the latency requirements?

*System Priorities:*
- What non-functional requirements are most critical?
- Can we tolerate some downtime or eventual consistency?
- Are there specific compliance or security requirements?

**Functional Requirements:**
- Define core functionalities (same as in LLD)
- Identify critical vs. nice-to-have features
- Determine user interactions and workflows

**Non-Functional Requirements:**
- Discuss CAP theorem implications
- Think through which aspect (Consistency, Availability, Partition Tolerance) is most crucial
- Speak your reasoning out loud
- Consider generic priorities: scalability, availability, and performance

**Skipping Capacity Estimation:**
- You can tell the interviewer you want to skip back-of-the-envelope estimation for now
- Return to it later if time permits or if needed for design decisions

---

## Back of the Envelope Calculation

### Key Metrics and Conversions

| Unit | Value | Conversion |
|------|-------|-----------|
| 1 KB | 1,000 bytes | - |
| 1 MB | 1,000 KB | 1,000,000 bytes |
| 1 GB | 1,000 MB | 1,000,000,000 bytes |
| 1 TB | 1,000 GB | 1,000,000,000,000 bytes |
| 1 Million | 1,000,000 | 10^6 |
| 1 Billion | 1,000,000,000 | 10^9 |

### Performance Benchmarks

| Operation | Approximate Time |
|-----------|------------------|
| L1 Cache Reference | 0.5 ns |
| L2 Cache Reference | 7 ns |
| Main Memory Reference | 100 ns |
| Read 1 MB sequentially from memory | 250 μs |
| Disk Seek | 10 ms |
| Read 1 MB sequentially from disk | 30 ms |
| Network Round Trip (same datacenter) | 0.5 ms |
| Realtime Latency Threshold | 200 ms |

### Database Performance

**SQL vs NoSQL Write Performance:**
- Calculate reads/writes per second requirements
- SQL databases: ~1,000-10,000 writes/sec per node (ACID overhead)
- NoSQL databases: ~10,000-100,000 writes/sec per node (eventual consistency)
- Use these benchmarks to justify database choice

**When to Calculate:**
- Storage requirements for different data types
- Read/write ratios to determine caching strategy
- Database sharding needs based on throughput limits
- Memory requirements for caching layers

### Estimation Strategy

1. Start with Daily Active Users (DAU)
2. Calculate requests per second (DAU × actions per user / 86400)
3. Consider peak traffic (multiply by 2-3x)
4. Estimate data size per entity
5. Calculate total storage (entities × size × growth rate)
6. Determine cache size (20/80 rule: cache 20% for 80% hits)

---

## API Design

### Time Allocation and Communication Protocol Selection

**Default Strategy:**
- Spend no more than 5 minutes on API Design unless specifically asked
- Focus on defining key endpoints quickly
- Save detailed discussion for deep dive if requested

**Communication Protocols:**
| Communication Type | Protocol | Rationale |
|-------------------|----------|-----------|
| Inter-microservice | RPC (gRPC) | High efficiency, binary protocol, optimized for internal communication |
| Client-facing APIs | HTTP/REST | Clients are diverse and well-versed with HTTP; universal compatibility |

**Why HTTP/REST for clients despite RPC efficiency?**
- Clients include web browsers, mobile apps, third-party integrations
- HTTP/REST is universally understood and supported
- Better debugging and tooling ecosystem
- Industry standard for external-facing APIs
- RPC reserved for internal microservice communication where efficiency is critical

### REST API Structure

**Core Principles:**
- Use nouns only, not verbs
- Use plural nouns for resource naming
- Follow RESTful conventions consistently

**Endpoint Naming Convention:**
- Use nouns, not verbs: `/users` not `/getUsers`
- Use plural nouns: `/products` not `/product`
- Use hierarchical structure: `/users/{userId}/orders`

**HTTP Methods:**
| Method | Purpose | Example |
|--------|---------|---------|
| GET | Retrieve resource(s) | `GET /users/{userId}` |
| POST | Create new resource | `POST /users` |
| PUT | Update entire resource | `PUT /users/{userId}` |
| PATCH | Partial update | `PATCH /users/{userId}` |
| DELETE | Remove resource | `DELETE /users/{userId}` |

**Path Parameters vs Query Parameters:**
- `:id` in path = required parameter: `/users/:userId`
- `?` in query = optional parameter: `/users?role=admin&status=active`

### API Design Best Practices

**Security:**
- Never include `userId` in API request body
- Extract user identity from JWT token on backend
- Use authentication headers for sensitive operations

**Request/Response Structure:**
```
POST /api/v1/orders
Authorization: Bearer <JWT_TOKEN>
Request Body:
{
  "productId": "123",
  "quantity": 2,
  "shippingAddress": { ... }
}
Response:
{
  "orderId": "456",
  "status": "pending",
  "createdAt": "2024-01-01T00:00:00Z"
}
```

**Status Codes:**
| Code | Meaning | Use Case |
|------|---------|----------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Missing/invalid auth |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server failure |
| 503 | Service Unavailable | Temporary downtime |

**Extended Protocols:**
| Protocol | Use Case | Characteristics |
|----------|----------|----------------|
| HTTPS/REST | Client-facing APIs | Request-response, stateless, universal support |
| gRPC (RPC) | Inter-microservices | High performance, binary protocol, efficient |
| WebSockets | Real-time bidirectional | Chat, live updates, gaming |
| SSE (Server-Sent Events) | Unidirectional streaming | Live feeds, notifications |
| AMQP/MQTT | Asynchronous messaging | Message queues, IoT |

**SSE (Server-Sent Events) Details:**
- Unidirectional communication (server to client)
- Uses HTTP protocol
- Built-in browser support for auto-reconnection
- Appears as a long-running HTTP connection
- Uses HTTP header to indicate chunked response
- Ideal for live feeds, notifications, and updates

### API Gateway Considerations
- **Data Size Limit:** API Gateway typically has ~10 MB limit
- For large files, use presigned URLs or direct upload to blob storage
- Implement rate limiting at gateway level
- Use for request routing to microservices

---

## Choosing the Right Database

...(REMAINS AS IN ORIGINAL)...