# Proximity Search Technologies & System Design

---

# Part 1: Core Technologies (SDE‑2 Friendly)

---

## Geohashing

### Concept
Geohashing converts a location `(lat, lon)` into a short string like `"tdr1k6"` so that nearby locations usually share a prefix. It is essentially a fixed grid laid over the world, encoded as strings.

### Why it is useful
* Easy to compute from GPS coordinates in application code.
* Can be stored as a normal string key in any database.
* Very good for **high‑frequency location updates** (drivers pinging every few seconds).
* Makes sharding simple: route by geohash prefix (e.g., all `"tdr1*"` go to the same shard).

### Limitations
* **Boundary issue:** Two nearby points on different cell edges land in different geohashes, so for radius queries you query the center cell + 8 neighbors.
* Fixed cell sizes mean dense city areas can become **hot cells** with many points.
* Works best for "circle around point" style queries, not complex polygons.

### Typical use
* Live location for drivers/riders in ride‑hailing.
* "Find restaurants within 5km" style lookups when data changes frequently.
* Implemented via Redis GEO commands or storing geohash strings in a key‑value / NoSQL store.

---

## Quadtree

### Concept
Quadtree also divides the map into squares, but it **adapts to density**: busy areas are split into many small squares; sparse areas stay as few large squares.

### Why it is useful
* Handles uneven distributions (big cities vs villages) better than a fixed grid.
* Still supports efficient "things near this point" or "things in this box" queries.

### Limitations
* More complex to implement and maintain than plain geohashing.
* Inserts and updates are a bit heavier because the tree may need to rebalance.

### Typical use
* Games / AR worlds (Pokémon GO‑like), map rendering, heatmaps where the server owns an in‑memory structure.

---

## R‑Tree (used by PostGIS and other spatial DBs)

### Concept
R‑Tree groups nearby objects (points or polygons) into rectangles and builds a tree of those rectangles. Databases use it under the hood for spatial indexes.

### Why it is useful
* Supports **points and shapes** (e.g., delivery zones, city boundaries).
* Great for precise questions like "is this point inside this zone?" or "what shapes intersect this area?".
* Comes with full database features: transactions, joins, backups, etc.

### Limitations
* Index maintenance is heavier, so it is not ideal for millions of location updates per second.
* Sharding across many machines is trickier than with geohashing.

### Typical use
* Store locations of restaurants, warehouses, or zones in **PostGIS** or similar.
* Use for fairly static or slowly moving data, plus complex spatial logic.

---

## Elasticsearch Geospatial (geo_point / geo_shape)

### Concept
Elasticsearch can index a document with both text fields and a location field, so you can do **text + proximity** searches in one query.

### Why it is useful
* Example query: "Italian restaurants with rating ≥ 4 within 3km of me".
* Very good for analytics and aggregations such as heatmaps.
* Built‑in horizontal scaling and replication.

### Limitations
* Eventual consistency by default (new writes may take ~1s to be visible).
* Typically higher latency than in‑memory options like Redis.

### Typical use
* Search‑heavy applications where users type text and also filter by location.

---

## MongoDB 2dsphere Index

### Concept
MongoDB lets you attach a GeoJSON location to a document and index it with a 2dsphere index, allowing `$near` and `$geoWithin` queries.

### Why it is useful
* You can keep **business data and location together** in one document.
* Works well when you already use MongoDB as your primary store.

### Limitations
* Not designed for extreme write rates like a dedicated in‑memory geospatial store.
* You still need a good shard key strategy at big scale.

### Typical use
* Orders, users, or stores stored in MongoDB with basic proximity features.

---

## Technology Selection Cheatsheet (Interview‑Level)

| Scenario | Good Choice | Why |
|---------|-------------|-----|
| Lots of moving objects (drivers, delivery agents) with frequent updates | **Redis + Geohash** | Very fast writes, easy sharding, simple radius queries |
| Mostly static data, need accurate zones / polygons | **PostGIS (R‑Tree)** | Strong spatial features, ACID, good for shapes and boundaries |
| Text search + location filters | **Elasticsearch** | Combine full‑text and geo queries, strong aggregations |
| Already using MongoDB for main data | **MongoDB 2dsphere** | Keep data + location in one place, easy dev experience |
| In‑memory spatial logic inside a service (games/AR) | **Quadtree** | Adapts to density, good for in‑process proximity checks |

---

# Part 2: Proximity Search System Design Quick Reference

---

## 1. The 1‑Minute Pitch
* **What it is:** Proximity search finds the nearest objects to a given location or everything within a radius (e.g., "drivers within 3km").
* **Mental Model:** Think of it as a search engine that understands distance instead of only text.
* **System Placement:** Usually implemented in a data layer (Redis, PostGIS, Elasticsearch, MongoDB) behind a service like `LocationService`.
* **When to think of it:** 
  * Real‑time location‑based apps (ride‑hailing, food delivery).
  * Store‑locator / "near me" features.
  * Geofencing such as "is this user inside a delivery zone?".

## 2. Core Fundamentals Cheat Sheet
* **Data Models:**
  * Points: `(lat, lon)` (drivers, stores, users).
  * Shapes: polygons (zones, districts, no‑fly areas).
* **Two main indexing strategies:**
  * **Grid style (Geohash):** fixed cells, very fast writes, easy sharding.
  * **Tree style (Quadtree / R‑Tree):** adaptive or grouped cells, better for complex queries and shapes.
* **Consistency & durability:**
  * Redis: in‑memory, usually eventual consistency via async replicas.
  * PostGIS / MongoDB: durable on disk, standard DB guarantees.
* **Scaling:**
  * Geohash: shard by prefix.
  * R‑Tree / 2dsphere: shard by region or use built‑in DB sharding.

## 3. Architecture & Mechanics (Simplified)

**3.1 Write Path**
* **Redis + Geohash:** Service receives `(lat, lon, driverId)` → computes geohash or uses `GEOADD` → periodic updates (e.g., every 3–5 seconds while moving, less often when idle).
* **PostGIS / MongoDB:** Service writes GeoJSON data into DB; spatial index updates automatically. Used mainly when data does not change every few seconds.

**3.2 Read Path**
* **Radius / nearest‑neighbor query:**
  * Given a user location, query the index to get candidates within a bounding area.
  * Filter by exact distance and sort to get the closest N.
* **Zone / polygon query:**
  * Check if a point lies inside one or more polygons (e.g., which delivery zone or city district).

**3.3 Scaling & High Availability (High‑Level)**
* Horizontal scaling by **sharding on region or geohash prefix**.
* Replicas for read scaling and failover.
* For boundary cases (points near shard borders), query a couple of neighboring shards.

**3.4 Operational Knobs**
* Update frequency for moving objects (higher for fast‑moving, lower for idle).
* Geohash precision (coarser for large‑radius queries, finer for city‑level).
* TTL for ephemeral locations (remove offline/closed drivers automatically).

## 4. Interview Use Cases

* **Ride‑hailing / food delivery:**
  * Redis geospatial for live driver locations and nearest‑driver matching.
  * PostGIS or MongoDB for relatively static restaurant / zone data.
* **Store locator:**
  * PostGIS or MongoDB 2dsphere for "stores within X km".
* **Search + location:**
  * Elasticsearch for "text + near me" queries.

**When to CHOOSE which:**
* **Redis + Geohash:** high write QPS, low latency, simple radius queries.
* **PostGIS:** complex shapes and strict correctness.
* **Elasticsearch:** text search with geo filters.
* **MongoDB:** moderate scale where data + location should live together.

**When to AVOID over‑engineering:**
* Fewer than ~10K points and low QPS → a normal DB with a simple scan + index can be acceptable.

## 5. Trade‑offs & Pitfalls (SDE‑2 Level)

* **Geohash boundary problem:** Always mention querying neighboring cells.
* **Stale locations:** Use TTLs and reasonable update intervals so you are not matching against old positions.
* **Hot regions:** Very dense downtown areas can overload one shard; mitigate by increasing geohash precision or splitting that region across more nodes.
* **Write vs query balance:**
  * Many writes, simple reads → Redis + geohash.
  * Fewer writes, complex reads (polygons, joins) → PostGIS / Elasticsearch.

## 6. The "Drop‑In" Interview Script
> **Proposing it:** "For proximity search we’ll store live driver locations in Redis using its geospatial support, and keep relatively static data like restaurant locations in PostGIS."
>
> **Explaining choice:** "Redis plus geohash gives us very fast writes and low‑latency radius queries, which we need because drivers send location every few seconds. PostGIS handles polygons and accurate distance checks for zones and store locations."
>
> **Owning trade‑offs:** "We accept slightly stale data and approximate cells in Redis because user experience cares more about speed than exact meter‑level accuracy."

## 7. One‑Minute Recap
* **Use proximity search** when your feature is fundamentally "find nearest X" or "is this inside that zone".
* **Pick the tech** based on write rate, need for shapes vs points, and whether you also need text search.
* **Key strength:** Lets you answer location‑based queries efficiently instead of brute‑forcing every point.
* **Key weakness:** You must think about boundaries, stale positions, and sharding by region to avoid hot spots.

---
