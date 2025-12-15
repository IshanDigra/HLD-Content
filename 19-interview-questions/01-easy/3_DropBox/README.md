# Dropbox/Google Drive - Cloud File Storage System

## 1. Requirements

### Functional Requirements
| Functional Requirement |
|------------------------|
| Users must be able to **upload files** from any device to cloud storage |
| Users must be able to **download files** from any device with proper authentication |
| Users must be able to **share files** with other users and manage permissions |
| System must **automatically synchronize** files across all user devices in near-real-time |

### Non-Functional Requirements
| Non-Functional Requirement | Target | Justification |
|---------------------------|--------|---------------|
| **Availability** | 99.99% | Prioritize availability over consistency (AP from CAP). Users can always upload/download even during partial failures |
| **Consistency** | Eventual | Acceptable for users in different regions to see stale file versions (seconds delay) |
| **Latency** | <500ms | Control-plane operations (metadata, sync) should be fast |
| **File Size Support** | Up to 50 GB | Must handle large media files and datasets |
| **Resumability** | 100% | Large uploads must be resumable after network interruptions |
| **Data Integrity** | 99.999999999% (11 nines) | Zero data loss—reliable recovery mechanisms required |
| **Scalability** | 500M users, 100M DAU | Horizontal scaling for growing user base |

#### CAP Theorem Targeting:
- **Metadata Operations**: Availability > Consistency (AP/EAP)
- **User-scoped Queries**: Strong consistency acceptable within region (CP)

---

## 2. Capacity Estimation & Constraints

### Assumptions
| Parameter | Value |
|-----------|-------|
| Total Users | 500 million |
| Daily Active Users (DAU) | 100 million |
| Avg files per user | 200 files |
| Avg file size | 100 KB |
| Files uploaded per user/day | 1 file |
| Avg upload size per user/day | 5 MB |
| Read:Write ratio | 5:1 |

### Storage Estimation
```
Total files = 500M users × 200 = 100B files
Total storage = 100B × 100KB = 10PB (initial)
Daily growth = 100M DAU × 5MB = 500TB/day
5-year projection = 10PB + (500TB × 365 × 5) ≈ 922PB ≈ 1EB
```

### Bandwidth Estimation
```
Upload bandwidth = 500TB/day ÷ 86,400s ≈ 47 Gbps
Download bandwidth = 500TB × 5 (ratio) ÷ 86,400s ≈ 235 Gbps
Total bandwidth ≈ 282 Gbps
```

### QPS Estimation
```
Requests per minute = 100M DAU × 5 = 500M
Average QPS = 500M ÷ 60 ≈ 8.3M QPS
Peak QPS (3x) ≈ 25M QPS
```

### Capacity Summary Table
| Resource | Estimation |
|----------|------------|
| **Storage (initial)** | 10 PB |
| **Storage (5-year)** | ~1 EB |
| **Bandwidth** | 282 Gbps total |
| **QPS (avg/peak)** | 8.3M / 25M |
| **App Servers** | ~1,173 (with 3x redundancy) |
| **Metadata Storage** | 170 TB (with indexes) |

---

## 3. Core Entities

### Database Schema

| Entity | Field Name | Type | Description/Constraint |
|--------|------------|------|------------------------|
| **FileMetadata** | file_id | UUID (PK) | Unique identifier (SHA-256 of content) |
| | name | VARCHAR(255) | User-visible filename |
| | mime_type | VARCHAR(100) | File content type |
| | size_bytes | BIGINT | Total file size |
| | owner_id | UUID (FK → User) | File owner |
| | s3_key | TEXT | Blob storage object key |
| | status | ENUM | uploading \| uploaded \| failed \| deleted |
| | created_at | TIMESTAMP | Creation time (UTC) |
| | updated_at | TIMESTAMP | Last modification time (indexed for sync) |
| | content_hash | CHAR(64) | SHA-256 fingerprint (deduplication) |
| **Chunks** | file_id | UUID (FK) | Parent file |
| | chunk_id | VARCHAR(64) | SHA-256 of chunk content |
| | part_number | INT | Sequential chunk index |
| | size_bytes | INT | Chunk size |
| | etag | VARCHAR(64) | S3-provided checksum |
| | status | ENUM | not_uploaded \| uploading \| uploaded |
| **SharedFiles** | user_id | UUID (PK part) | User with access |
| | file_id | UUID (PK part, FK) | Shared file |
| | permission | ENUM | read \| write \| owner |
| | shared_at | TIMESTAMP | When sharing occurred |
| **User** | user_id | UUID (PK) | Unique user identifier |
| | email | VARCHAR(255), UNIQUE | User email |
| | password_hash | VARCHAR(255) | Bcrypt hashed password |
| | created_at | TIMESTAMP | Account creation time |

#### Relationships
- **User → FileMetadata**: 1:N (one user owns many files)
- **FileMetadata → Chunks**: 1:N (one large file has many chunks)
- **User ↔ FileMetadata (via SharedFiles)**: M:N (many users access many files)

#### Key Design Choices
- **UUID v4** for distributed ID generation without coordination
- **File ID = SHA-256** of content enables content-addressable storage and deduplication
- **Indexes**: (owner_id, updated_at), (file_id), (user_id, file_id)

---

## 4. API Design

| Method | Endpoint | Request | Response | Description |
|--------|----------|---------|----------|-------------|
| **POST** | /files/presigned-url | Body: name, mime_type, size_bytes, chunks[] | { file_id, upload_id, presigned_urls[] } | Initialize upload, return S3 pre-signed URLs |
| **PATCH** | /files/{fileId}/chunks | Body: chunk_id, part_number, status, etag | 200 OK | Update chunk status after S3 upload |
| **POST** | /files/{fileId}/complete | None | { status, s3_key } | Finalize multipart upload, mark file uploaded |
| **GET** | /files/{fileId}/presigned-url | Query: download=true | { download_url, metadata } | Generate pre-signed download URL |
| **GET** | /files/{fileId} | None | FileMetadata object | Fetch file metadata only |
| **POST** | /files/{fileId}/share | Body: user_ids[], permission | 200 OK | Share file with users |
| **GET** | /files/changes | Query: since={timestamp} | { files[], cursor } | Get changes since timestamp for sync |
| **DELETE** | /files/{fileId} | None | 204 No Content | Soft-delete file |

**Authentication**: JWT Bearer token in Authorization header (contains user_id for access control)

---

## 5. Data Flow

### Read Path: File Download
1. Client → GET /files/{fileId}/presigned-url
2. **File Service** checks permissions (SharedFiles table)
3. If authorized → Generate S3 pre-signed URL (time-limited, HMAC-signed)
4. Return download_url to client
5. Client downloads directly from S3 (bypassing app servers)

### Write Path: File Upload (Large File)
1. Client computes file SHA-256 and chunk fingerprints
2. POST /files/presigned-url → Backend creates FileMetadata (status="uploading"), initiates S3 multipart upload
3. Backend returns presigned URLs for each chunk
4. Client uploads chunks directly to S3 in parallel
5. For each chunk success → PATCH /files/{fileId}/chunks (backend verifies with S3 HEAD request)
6. After all chunks uploaded → POST /files/{fileId}/complete
7. Backend calls S3 CompleteMultipartUpload, updates status="uploaded"

### Sync Flow: Remote → Local
1. Client polls: GET /files/changes?since=<last_sync_timestamp>
2. Backend queries FileMetadata WHERE updated_at > timestamp AND (owner_id = user OR file_id IN SharedFiles)
3. Returns changed files list
4. Client compares with local DB:
  - New files → Download from S3
  - Updated files → Download changed chunks only (delta sync)
  - Deleted files → Remove locally

---

## 6. High-Level Design

### 6.1 Architecture Diagram

![Drop Box Architecture](../../Images/DropBox_GoogleDrive.excalidraw.svg)

### 6.2 Component Breakdown

| Component | Role/Responsibility |
|-----------|---------------------|
| **Client** | Desktop/mobile/web app; local folder watcher, sync daemon, chunking engine |
| **API Gateway** | SSL termination, JWT auth, rate limiting, request routing |
| **File Service** | Metadata CRUD, generate presigned S3 URLs, access control checks |
| **Sync Service** | Query changed files for sync, return delta lists |
| **Metadata DB** | DynamoDB/PostgreSQL—stores FileMetadata, Chunks, SharedFiles, Users |
| **Blob Storage (S3)** | Stores actual file content; supports multipart upload, presigned URLs, 11-nines durability |
| **Cache (Redis)** | Cache frequently accessed metadata, reduce DB load |
| **Message Queue** | (Optional) Pub/sub for real-time sync notifications via WebSocket |

**Load Balancing**: Round-robin with health checks  
**Storage Choice**: S3 for 11-nines durability, automatic replication across AZs  
**Scaling**: Horizontal scaling of stateless services, metadata DB sharded by user_id

---

## 7. Deep Dives

### Large File Support: Chunking + Multipart Upload

**Problem**: 50GB files exceed API limits, uploads fail on network interruptions

**Solution**: Chunking + S3 Multipart Upload + Presigned URLs

**Algorithm**:
```
1. Client splits file into 5-10MB chunks, computes SHA-256 per chunk
2. Client requests presigned URLs: POST /files/presigned-url
3. Backend initiates S3 multipart upload → Returns presigned URL per chunk
4. Client uploads chunks directly to S3 in parallel
5. Client notifies backend after each chunk: PATCH /files/{fileId}/chunks
6. Backend verifies via S3 HEAD request, stores ETag
7. After all chunks done: POST /files/{fileId}/complete
8. Backend calls S3 CompleteMultipartUpload (atomic assembly)
9. FileMetadata.status = "uploaded"
```

**Benefits**:
- No file data through app servers (offload to S3)
- Parallel chunk upload (faster)
- Resumable (retry failed chunks only)
- Progress tracking per chunk

---

### File Synchronization Algorithm

**Two Directions**:
1. **Local → Remote**: File watcher detects changes → Upload modified chunks
2. **Remote → Local**: Poll /files/changes → Download new/updated files

**Conflict Resolution**: Last Write Wins (LWW)
- File with latest updated_at wins
- Loser saved as "filename (conflicted copy).ext"

**Delta Sync**:
```
1. Client recomputes chunk fingerprints after edit
2. Compares with cached fingerprints
3. Uploads only changed chunks
4. Backend updates chunk metadata
```

**Efficiency**: For 10GB file with 2 changed chunks → Upload only 20MB (99.8% savings)

---

### Scaling Optimizations

**1. Sharding**: Metadata DB sharded by user_id (all user files on same shard)

**2. Caching**: Redis cache for hot metadata (cache-aside pattern, TTL=5min)

**3. Parallel Upload**: Client uploads 4 chunks concurrently (HTTP/2 multiplexing)

**4. Compression**: Gzip for text files (30-70% reduction), skip for media (already compressed)

**5. CDN (Optional)**: CloudFront for frequently downloaded public files (not critical for private user files)

---

### Failure Modes & Handling

| Failure | Mitigation |
|---------|------------|
| **DB Master Down** | Auto-failover to replica (30-60s downtime), queue writes during failover |
| **S3 Outage** | Cross-region replication, switch to replica region, exponential backoff retry |
| **Cache Stampede** | Request coalescing, circuit breaker, pre-warm hot keys |
| **Network Partition** | Serve stale cached data (mark with staleness header), queue writes |

---

### Security

**Encryption**:
- **In Transit**: TLS 1.3 for all client-server communication
- **At Rest**: S3 SSE-KMS (server-side encryption with customer keys)

**Authentication**: JWT with short-lived access tokens (15min), refresh tokens for renewal

**Access Control**: Every file operation checks owner_id or SharedFiles permissions

**Presigned URL Security**: Time-limited (1 hour), HMAC-signed, scoped to specific S3 object

**Rate Limiting**: Per-user/IP limits at API Gateway (prevent abuse)

---

### Telemetry

**Key Metrics**:

| Metric                  | Threshold | Action             |
|-------------------------|-----------|--------------------|
| Upload success rate     | <95%      | Alert on-call      |
| Download latency (p99)  | >2s       | Investigate S3/CDN |
| Sync lag                | >60s      | Check Sync Service |
| DB read latency (p95)   | >50ms     | Scale replicas     |


**Structured Logging** (JSON):
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "service": "file-service",
  "user_id": "uuid-123",
  "file_id": "uuid-456",
  "action": "chunk_upload_verified",
  "latency_ms": 45
}
```

**Distributed Tracing**: OpenTelemetry traces across Client → API Gateway → File Service → S3 → DB

---

## 8. References

**Visual**: [DropBox_GoogleDrive.excalidraw.jpg](DropBox_GoogleDrive.excalidraw.jpg)  
**Video**: [Design Dropbox w/ Ex-Meta Staff Engineer](https://www.youtube.com/watch?v=_UZ1ngy-kOI&t=2032s)  
**Deep Dive**: [HelloInterview - Dropbox Design](https://www.hellointerview.com/learn/system-design/problem-breakdowns/dropbox)

---

**Document Version**: 1.0  
**Last Updated**: November 27, 2025