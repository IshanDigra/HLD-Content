# Uber System Design

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
- [ ] Users input pickup and drop-off locations to get fare estimations.
- [ ] Users can request a ride.
- [ ] System matches ride requests with available drivers nearby.
- [ ] Drivers can accept or reject ride requests.
- [ ] *Out of Scope*: Different vehicle types, payments, ratings, chat/calling, advance scheduling.

### Non-Functional Requirements (SPARCS)
- [ ] **Scalability**: High throughput in terms of concurrent ride requests and driver location updates.
- [ ] **Low Latency**: Request matching (< 1 min), near real-time fare estimation and location tracking.
- [ ] **Consistency**: Highly consistent in ride matching (1 Rider matched to exactly 1 Driver). No double bookings.
- [ ] **Availability**: High availability for the system as a whole.
- [ ] *Out of Scope*: Privacy/security data, deep resiliency/failover mechanics (handled by standard cloud practices).

---

## 2. Core Entities (3-5 min)

- **Rider**: `riderId`, `name`, `contactInfo`
- **Driver**: `driverId`, `vehicleId`, `status` (Available, Busy, Offline), `currentLocation` (Lat/Long)
- **Ride / Trip**: `tripId`, `riderId`, `driverId`, `pickupLoc`, `dropLoc`, `status`, `fare`
- **Relationships**:
  - Rider requests many Trips.
  - Driver completes many Trips.

---

## 3. API Design (~5 min)

### `GET /api/v1/fare-estimate`
- **Purpose**: Get estimated fare before booking.
- **Request Parameters**: `pickup_lat`, `pickup_long`, `drop_lat`, `drop_long`
- **Response**: `200 OK` with estimated amount.

### `POST /api/v1/ride`
- **Purpose**: Request a new ride.
- **Request Body**: `{ "riderId": "123", "pickupLoc": {...}, "dropLoc": {...} }`
- **Response**: `202 Accepted` with `tripId` (Async matching starts).

### `PUT /api/v1/driver/location`
- **Purpose**: Update driver's real-time location.
- **Request Body**: `{ "driverId": "456", "lat": 37.77, "long": -122.41 }`
- **Response**: `200 OK`

---

## 4. Data Flow (5-10 min)

1. Driver app sends location updates to Gateway every few seconds.
2. `Location Service` updates the Spatial Database/Cache (e.g., Redis Geospatial).
3. Rider requests a ride. Hits the `Ride Service`.
4. `Ride Service` calls `Location Service` to query nearby available drivers.
5. `Match Service` ranks drivers (ETA, direction) and pushes notification to the best Driver via `Notification Service`.
6. Driver accepts. State updates to `Busy`. Notification sent back to Rider.

---

## 5. High-Level Design (15-20 min)

- **Clients**: Rider App, Driver App.
- **API Gateway**: Load balancing, routing, auth.
- **Location Service**: Ingests high-throughput driver location updates and powers geospatial queries.
- **Ride Service**: Manages the Trip lifecycle (Requested -> Matched -> In-Progress -> Completed).
- **Match Service / Dispatcher**: Complex logic to find the optimal driver based on distance, traffic, and direction.
- **Notification Service**: Real-time push notifications using APN (iOS) and FCM (Android) or WebSockets.
- **Databases**:
  - **Relational DB**: Postgres for Users, Trips, and Payments (ACID properties for trip state).
  - **Geospatial DB**: Redis Geo (for fast in-memory proximity searches) or Postgres with PostGIS plugin.

---

## 6. Deep Dives (15-20 min)

### Location Tracking & Geospatial Indexing
- **Challenge**: Storing and querying millions of lat/long coordinates every 3-5 seconds.
- **Solution**: Geo-hashing (dividing the map into a grid of characters, e.g., Quadtree or Geohash). Redis provides built-in `GEOADD` and `GEORADIUS` commands which are incredibly fast for "drivers near me" queries.
- **Trade-off**: Redis data might be lost on crash. Solution is to asynchronously flush driver locations to a persistent store (Cassandra or PostGIS) for analytics and historical paths.

### Concurrency and Ride Matching
- **Challenge**: Two riders requesting a ride at the same time could be assigned the same driver.
- **Solution**: The Match Service must use distributed locks (Redis Redlock) or atomic database operations (e.g., `UPDATE Driver SET status = 'Busy' WHERE driverId = 123 AND status = 'Available'`) to ensure 1 Rider maps to 1 Driver.

---

## 7. Address Key Issues (5 min)

### Fault Tolerance & Resiliency
- WebSockets for notifications can drop. The App must fall back to long-polling or polling if the connection drops.
- Microservices must handle retry logic gracefully for intermittent network failures.

### Monitoring & Observability
- Track matching time, driver acceptance rate, and location update latency.
