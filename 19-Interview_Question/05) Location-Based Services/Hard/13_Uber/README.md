# Design Uber

## Problem Statement
Design a ride-sharing service like Uber where users can book rides and drivers can accept them. The system needs to handle real-time location updates, efficient driver matching, and high concurrency.

---

## 1. High Level Design

### 1. Functional Requirements
- **Riders**: Book a ride, view estimated fare, track driver location.
- **Drivers**: Accept/Decline rides, navigate to pickup/drop-off, update location.
- **System**: Match riders with nearby drivers, calculate fares, handle payments.

### 2. Non-Functional Requirements
- **Scalability**: Handle high throughput (millions of users).
- **Latency**: Low latency for matching (< 1 min) and location updates.
- **Availability**: High availability for booking service.
- **Consistency**: Strong consistency for ride status (no double booking).
- **Fault Tolerance**: Resilient to server/network failures.

### 3. Capacity Estimation
| Metric | Value |
|---|---|
| DAU | 100 Million |
| Rides/Day | 10 Million |
| Write QPS (Location Updates) | ~200k/sec (assuming 500k active drivers updates every 5s) |
| Storage (5 Years) | ~500 TB (Ride history + Location logs) |

---

## 2. Core Entities

| Entity | Attributes |
|---|---|
| **Rider** | `ID`, `Name`, `Email`, `Rating`, `PaymentInfo` |
| **Driver** | `ID`, `Name`, `VehicleDetails`, `Rating`, `Status` (Available/Busy) |
| **Ride** | `ID`, `RiderID`, `DriverID`, `Source`, `Destination`, `Status`, `Fare` |
| **Location** | `DriverID`, `Latitude`, `Longitude`, `Timestamp` |

---

## 3. API Design

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v1/rides` | Request a new ride |
| `POST` | `/api/v1/driver/location` | Update driver location (Heartbeat) |
| `GET` | `/api/v1/rides/{id}` | Get ride status/details |
| `POST` | `/api/v1/rides/{id}/accept` | Driver accepts a ride |

---

## 4. Database Schema

### Rider Table
| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `name` | VARCHAR | NOT NULL |
| `email` | VARCHAR | UNIQUE |

### Driver Table
| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `status` | ENUM | Available, Busy, Offline |

### Ride Table
| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `rider_id` | UUID | FK |
| `driver_id` | UUID | FK, Nullable |
| `status` | ENUM | Requested, Matched, Started, Completed |

---

## 5. High Level Architecture

### 5.1 Architecture Diagram
![Uber Architecture](../../../../19-interview-questions/Images/Uber.excalidraw.svg)

### 5.2 Component Breakdown
- **Load Balancer**: Distributes traffic (Round Robin/Least Connections).
- **API Gateway**: Auth, Rate Limiting, Routing.
- **Ride Service**: Handles booking logic.
- **Driver Service**: Manages driver status and profiles.
- **Location Service**: Ingests high-frequency location updates (Redis/Geospatial DB).
- **Matching Service**: Algorithms to pair riders with drivers.
- **Notification Service**: Push notifications (FCM/APNS) for updates.

---

## 6. Data Flow (Request Lifecycle)
1. **Ride Request**: Rider app -> LB -> API Gateway -> Ride Service.
2. **Matching**: Ride Service -> Matching Service -> Queries Spatial Index (Redis Geo).
3. **Notification**: Matching Service -> Notification Service -> Driver App.
4. **Acceptance**: Driver accepts -> Ride Service updates DB (Locking) -> Notify Rider.

---

## 7. Deep Dives & Optimizations

### 1. Geospatial Indexing (QuadTree / Geohash)
- **Problem**: Finding nearby drivers efficiently.
- **Solution**: Use **Geohash** (strings) or **QuadTree** (hierarchical grids).
- **Tech Choice**: **Redis Geo** (Geohash based) for fast lookups.

### 2. Handling High Write Throughput (Location Updates)
- Drivers send location every ~5 seconds. Direct DB writes will choke.
- **Optimization**: Update **Redis** (Transient) -> Flush to **Cassandra/S3** (Persistent History) asynchronously.

### 3. Distributed Locking (Double Booking)
- **Problem**: Multiple drivers accepting same ride or vice-versa.
- **Solution**: Use **Redis Distributed Lock (Redlock)** or Database Optimistic Locking (`version` column) during assignment.

### 4. Matching Algorithm
- Simple: Nearest available driver.
- Advanced: Batch matching (Maximize global utility/minimize total wait time) using **Bipartite Matching** or **Hungarian Algorithm**.

---

## 8. References
- **Visual**: ![Uber HLD](./Uber.excalidraw.jpg)
- **Video**: [System Design Interview - Uber](https://www.youtube.com/watch?v=lsKU38RKQSo)
- **Deep Dive**: [Hello Interview - Uber Breakdown](https://www.hellointerview.com/learn/system-design/problem-breakdowns/uber)
