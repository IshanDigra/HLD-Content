# DropBox/GoogleDrive System Design

## Table of Contents
- [1. Requirements (5-10 min)](#1-requirements-5-10-min)
- [2. Core Entities (3-5 min)](#2-core-entities-3-5-min)
- [3. API Design (~5 min)](#3-api-design-5-min)
- [4. Data Flow (5-10 min)](#4-data-flow-5-10-min)
- [5. High-Level Design (15-20 min)](#5-high-level-design-15-20-min)
- [6. Deep Dives (15-20 min)](#6-deep-dives-15-20-min)
- [7. Address Key Issues (5 min)](#7-address-key-issues-5-min)

---
## 1. Requirements (5-10 min)

### Functional Requirements
- [ ] Users should be able to upload files from any device.
- [ ] Users should be able to download files from any device.
- [ ] Users should be able to share files with other users and view files shared with them.
- [ ] Users should be able to automatically sync files across devices.
- [ ] *Out of scope:* Editing files, viewing files without downloading, blob storage internal design.




### Back-of-the-Envelope (BOE) Calculations
**Step 1: Assumptions**
- Users: 500M MAU -> 100M DAU
- Activity: 5 files uploaded/user/day, 5 downloaded
- Payload: Average file size ~1 MB (Chunked into 4MB blocks)

**Step 2: Load (QPS)**
- Write QPS: (100M * 5) / 100,000 ≈ 5,000 QPS
- Read QPS: 5,000 QPS

**Step 3: Storage (5-year plan)**
- Daily Storage: 5,000 QPS * 100,000s * 1 MB = 500 TB/day
- 5-year storage: 500 TB * 365 * 5 ≈ 900 PB

**Step 4: Bandwidth**
- Egress: 5,000 QPS * 1 MB ≈ 5 GB/s
- Ingress: 5,000 QPS * 1 MB ≈ 5 GB/s

**Step 5: Cache**
- Metadata cache for fast sync checks: 20% of active files metadata.

### Non-Functional Requirements
- [ ] **Scalability**: Ability to handle large files efficiently.
- [ ] **Performance**: Low latency for uploading and downloading.
- [ ] **Availability**: High availability for downloading and uploading files.
- [ ] **Reliability**: Zero data loss for files uploaded.
- [ ] **Consistency**: Eventual consistency for file sync across devices.
- [ ] *Out of scope:* User storage limits, file versioning, virus scanning.

---

## 2. Core Entities (3-5 min)

- **User**: `userId`, `name`, `email`
- **File**: `fileId`, `s3Url`, `size`, `checksum`
- **FileMetadata**: `metadataId`, `fileId`, `filename`, `path`, `userId`, `lastModified`
- **Folder**: `folderId`, `name`, `parentId`, `userId`
- **Relationships**:
  - User owns many Files and Folders.
  - Folders can contain Files or other Folders.

---

## 3. API Design (~5 min)

### `POST /Files`
- **Purpose**: Upload a new file.
- **Headers**: `Authorization: Bearer <JWT>` (determines userId)
- **Request Body**: File binary and FileMetadata.
- **Response**: `200 OK` or `201 Created` with File ID.

### `GET /Files/{fileId}`
- **Purpose**: Download file and fetch metadata.
- **Response**: `200 OK` with File bytes and Metadata.

### `POST /Files/Share`
- **Purpose**: Share a file with other users.
- **Headers**: `Authorization: Bearer <JWT>`
- **Request Body**: `{ "fileId": "<id>", "users": ["userId1", "userId2"] }`
- **Response**: `200 OK`

---

## 4. Data Flow (5-10 min)

### Upload & Sync Flow
1. Client breaks large files into smaller chunks.
2. Client sends request to API Gateway.
3. Gateway routes to `Upload Service`.
4. File chunks are uploaded directly to Object Storage (S3).
5. Metadata is committed to Metadata DB.
6. A notification event is fired to the `Sync Service` via a Message Queue.
7. `Sync Service` pushes updates to the user's other connected devices via WebSockets.

---

## 5. High-Level Design (15-20 min)

### High-Level Architecture
```mermaid
graph TD
    A[Client] --> B[Block Server]
    B --> C{Chunk Hasher}
    C -->|Hash Exists| D[Metadata Update Only]
    C -->|New Hash| E[Upload to S3]
    F[Sync Service] --> G[[Message Queue]]
    G --> H[WebSocket Notifier]
```




- **Client Application**: Desktop/Mobile app running a sync agent.
- **API Gateway**: Load balancing, auth verification (JWT).
- **Block Servers**: Servers that handle splitting files into blocks, hashing, and uploading blocks to storage.
- **Metadata Servers**: Manage file paths, permissions, and folder structures. Store data in a relational DB (ACID properties for metadata).
- **Object Storage (S3)**: Stores raw file blocks.
- **Message Queue (Kafka)**: Handles async events for syncing and notifications.
- **Notification Service**: Maintains WebSocket connections with clients for real-time sync pushes.

---

## 6. Deep Dives (15-20 min)

### Deep Dive / Data Flow
```mermaid
sequenceDiagram
    participant Client
    participant API_Gateway
    participant Service
    participant Database

    Client->>API_Gateway: Request
    API_Gateway->>Service: Route
    Service->>Database: Query/Update
    Database-->>Service: Result
    Service-->>API_Gateway: Response
    API_Gateway-->>Client: Result
```


### Block Storage & Chunking
> **Challenge**: Uploading a 10GB file natively takes too long, consumes massive memory, and if the network drops at 99%, the entire upload fails.
>
> **Solution**: The client application breaks files into smaller blocks (e.g., 4MB chunks).
> - Blocks are uploaded in parallel to the `Block Servers`.
> - The `Metadata DB` stores an ordered array of `block_ids` for a given file.
> - **Trade-offs**: Increases metadata complexity significantly, but allows resumable uploads and delta syncs.

### Delta Sync & Deduplication
> **Challenge**: Minimizing bandwidth and storage when thousands of users upload the exact same viral video, or when a user changes just one sentence in a 1GB text file.
>
> **Solution**:
> - **Delta Sync**: If a user modifies a file, the client only hashes and uploads the *modified* 4MB blocks, not the entire file.
> - **Deduplication**: Calculate a SHA-256 hash for every block. Before uploading, the client asks the metadata server if the hash exists. If multiple users upload the exact same block, we store only one physical copy in S3 and reference it multiple times in the DB.

## 7. Address Key Issues (5 min)

### Fault Tolerance & Resiliency
- **ACID Metadata**: The Metadata DB needs strict ACID guarantees (e.g., PostgreSQL or Spanner) because file permissions, paths, and block sequences cannot be eventually consistent without causing severe user corruption. Use Leader-Follower replication.
- **S3 Durability**: Object storage natively replicates across multiple AZs offering 99.999999999% durability.

### Security
- Files must be encrypted at rest in S3 using AES-256.
- Strict authorization checks in Metadata servers before granting S3 Pre-signed URLs for block downloads.

### Key Concepts on the Go
- **Checksumming**: Essential for verifying data integrity. The client hashes the file, and the server verifies it upon receipt to ensure bits weren't flipped during transit.
- **Long Polling vs WebSockets**: To notify clients of changes, WebSockets provide true real-time, bi-directional communication, while Long Polling is a fallback for restrictive corporate firewalls.

## References & Original Diagrams
![DropBox_GoogleDrive Architecture](../../../../19-interview-questions/Images/DropBox_GoogleDrive.excalidraw.svg)
