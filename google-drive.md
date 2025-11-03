# Dropbox / Google Drive System Design

## Overview

Design a cloud-based file storage service supporting upload, download, sharing, and automatic synchronization across devices.

## Requirements

### Functional

- Upload files from any device
- Download files from any device  
- Share files with other users
- Automatic file synchronization across devices

### Non-Functional

- High availability (prioritize availability over consistency)
- Support files up to 50GB
- Low latency for uploads, downloads, and syncing
- Secure and reliable (file recovery if lost/corrupted)

**CAP Theorem Note**: Unlike stock trading where immediate consistency is critical, file storage can tolerate eventual consistency. If a user in Germany uploads a file, it's acceptable if a user in the US sees it a few seconds later.

## Core Entities

- **File**: Raw bytes of uploaded content
- **FileMetadata**: Name, size, MIME type, uploader, chunks, status
- **User**: System user

## API Design

```
POST /files/presigned-url { FileMetadata } -> { presignedUrl }
GET /files/{fileId} -> FileMetadata
POST /files/{fileId}/share { User[] } 
GET /files/{fileId}/changes -> FileMetadata[]
```

User authentication passed via headers (session token or JWT) for security.
## High-Level Design

### 1. Upload Files

**Where to store?**
- **File Metadata**: NoSQL database like DynamoDB (loosely structured, simple queries by user)
  - Schema: `{ id, name, size, mimeType, uploadedBy, status, chunks[] }`
- **File Content**: Blob storage (S3)

**Upload Approaches**:

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Upload to Server** | Client → API Server → S3 | Simple | Slow (2 trips), wastes server resources |
| **Direct Upload to S3** ✅ | Client → S3 (via presigned URL) | Fast, scalable | Slightly complex |

**Pattern: Handling Large Blobs**

Upload large files directly to S3 using **presigned URLs**:
1. Client requests presigned URL from File Service
2. File Service generates signed URL (valid for ~15 min) using S3 SDK
3. Client uploads directly to S3 using presigned URL
4. S3 triggers event notification → Lambda updates FileMetadata

**Benefits**: Bypasses application servers, reduces latency, saves server resources.

### 2. Download Files

**Download Approaches**:

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Server Proxy** | Client → API Server → S3 → API Server → Client | Simple | Slow, wastes bandwidth |
| **Direct Download** ✅ | Client → S3 (via presigned URL) | Fast, scalable | Requires signed URLs |
| **CDN Caching** ✅✅ | Client → CDN → S3 (cache miss) | Lowest latency | Cost for popular files |

**Recommended**: Direct S3 download + CDN for popular files.

### 3. File Sharing

**Implementation**:
- Store `sharedWith: [userId]` in FileMetadata (or separate ShareList table)
- On download request, check if user is owner or in `sharedWith` list
- Generate presigned URL only for authorized users
- Presigned URLs expire (5-15 min) to prevent unauthorized sharing

**Access Control List (ACL)**:
- Simple permissions model: owner + shared users
- Could extend to: view-only, edit, comment permissions

### 4. Automatic Sync Across Devices

**Two-Way Sync**:

**Local → Remote (Push Changes)**:
- Client-side **sync agent** monitors local Dropbox folder using OS file system events (FSEvents on macOS, FileSystemWatcher on Windows)
- Detects changes → queues for upload → uploads to S3 → updates FileMetadata
- Conflict resolution: **Last Write Wins** (most recent edit saved)
- Note: Versioning would add new file versions instead of overwriting

**Remote → Local (Pull Changes)**:
- Client needs to know when remote changes occur

**Sync Strategies**:

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Polling** | Periodically ask "any changes since last sync?" | Stale files (not edited recently) |
| **WebSocket/SSE** | Server pushes real-time notifications | Fresh files (recently active) |
| **Hybrid** ✅ | WebSocket for fresh files, polling for stale | Optimal resource usage |

**Hybrid Approach**:
- Fresh files (edited within last few hours): WebSocket for real-time sync
- Stale files (inactive): Periodic polling to conserve resources

## System Architecture

**Components**:

- **Uploader/Downloader**: Client apps (web, mobile, desktop) with sync agent
- **Load Balancer + API Gateway**: SSL termination, rate limiting, request routing
- **File Service**: Generates presigned URLs, manages FileMetadata (no direct file handling)
- **File Metadata DB**: DynamoDB storing file metadata and shared files table
- **S3**: Actual file storage (with encryption at rest)
- **CDN**: Caches popular files close to users for low latency

## Deep Dives

### 1. Supporting Large Files (50GB)

**User Experience Requirements**:
- ✅ Progress indicator during upload
- ✅ Resumable uploads (survive interruptions)

**Why Single POST Fails**:
- Timeouts: 50GB @ 100Mbps = 1.11 hours (exceeds server/browser timeouts)
- Size limits: API Gateway default = 10MB max payload
- Network interruptions: Must restart from scratch
- No progress visibility

**Solution: Chunking**

Break files into 5-10MB chunks on **client-side** before upload:

**Implementation**:
1. **Client chunks file** + calculates fingerprint (SHA-256) per chunk and entire file
   - Fingerprint = unique content hash (not filename-based)
   - FileId = fingerprint of entire file

2. **Check if file exists**: `GET /files/{fileId}` (resume existing upload)

3. **Request presigned URL**: `POST /files/presigned-url` with metadata
   - Backend saves FileMetadata with `status: "uploading"` and chunks array:
   ```json
   {
     "id": "file-fingerprint-123",
     "name": "video.mp4",
     "size": 50000000000,
     "chunks": [
       { "id": "chunk-fingerprint-1", "status": "not-uploaded" },
       { "id": "chunk-fingerprint-2", "status": "not-uploaded" }
     ]
   }
   ```

4. **Upload chunks in parallel** to S3 using multipart upload

5. **Track progress**: After each chunk upload, send `PATCH /files/{fileId}/chunks/{chunkId}` with ETag
   - Backend verifies with S3 (HEAD request or ListParts API)
   - Updates chunk status to "uploaded"

6. **Complete upload**: When all chunks = "uploaded", mark file status = "uploaded"

**S3 Multipart Upload**:
- AWS provides built-in multipart upload API
- JavaScript SDK handles chunking/uploading automatically
- Event notifications trigger only when entire upload completes (not per part)
- Use ListParts API or HEAD requests to verify individual chunk uploads

**Benefits**:
- Parallel uploads maximize bandwidth utilization
- Resume from exact chunk that failed (via fingerprints)
- Adaptive chunk sizes based on network conditions
- Progress tracking for better UX

### 2. Optimizing Speed (Upload/Download/Sync)

**Techniques**:

**For Download**:
- ✅ **CDN**: Cache files close to users (reduces distance data travels)
- ✅ **Chunking**: Download in parallel chunks to maximize bandwidth

**For Upload**:
- ✅ **Parallel chunk uploads**: Maximize bandwidth utilization
- ✅ **Adaptive chunk sizes**: Adjust based on network conditions
- ✅ **Compression**: Reduce bytes transferred (smart application)

**For Sync**:
- ✅ **Delta sync**: Only sync changed chunks (not entire file)
- ✅ **Hybrid push model**: WebSocket for active files, polling for stale

**Compression Trade-offs**:
- Worthwhile when: `time_saved(fewer bytes) > time_spent(compress + decompress)`
- **Text files**: 5GB → 1GB+ compression (worth it!)
- **Media files** (PNG, MP4): ~5% compression (not worth it)
- **Decision logic**: Client determines based on file type, size, network speed
- **Algorithms**: Gzip (universal), Brotli (better ratio), Zstandard (best ratio + speed)
- **Important**: Compress BEFORE encrypting (encryption adds randomness, making compression ineffective)

### 3. File Security

**Encryption in Transit**:
- HTTPS for all client-server communication

**Encryption at Rest**:
- S3 server-side encryption enabled
- S3 encrypts files with unique keys, stored separately from file data

**Access Control**:
- `sharedWith` array or separate ShareList table = basic ACL
- Presigned URLs only for authorized users

**Preventing Unauthorized Sharing**:
- Problem: Authorized user shares download link publicly (forum, social media)
- Solution: **Signed URLs with expiration** (5-15 min validity)
  - Generated on server with secret key
  - CDN (CloudFront) validates signature + expiration on each request
  - Expired or invalid URLs = access denied

**Signed URL Flow**:
1. **Generation**: Server creates URL with signature (path + expiration + secret key)
2. **Distribution**: Authorized user receives signed URL
3. **Validation**: CDN verifies signature + timestamp on request
4. **Access**: If valid & not expired → serve file, else → deny

## Key Learnings & Insights

### Design Patterns

**Pattern: Handling Large Blobs**
- Upload directly to blob storage (S3) using presigned URLs
- Bypass application servers to save resources and reduce latency
- Multipart/chunked uploads for reliability and resumability
- Applies to: Any large file system (photos, videos, backups, documents)

**Chunking is Critical**:
- Not just for resumability, but for speed (parallel uploads/downloads)
- Must happen on **client-side** (defeats purpose if server chunks after receiving full file)
- Chunk size: 5-10MB (balance between overhead and granularity)
- Fingerprinting enables deduplication + accurate resume detection

**Presigned URLs**:
- Security + performance in one pattern
- Client uploads/downloads directly without server proxy
- Time-limited access (15 min typical) prevents URL sharing exploits
- Works with CDNs (CloudFront validates signatures)

### Storage Trade-offs

**Metadata vs File Content**:
- Metadata: DynamoDB/Postgres (structured, queryable, small size)
- File content: S3 (blob storage, massive scale, cheap)
- Different access patterns → different storage solutions

**Database Choice**:
- NoSQL (DynamoDB): Flexible schema, horizontal scaling, simple queries
- SQL (Postgres): Also works! ACID guarantees, complex joins
- Key insight: For this use case, either works—choice less critical than understanding trade-offs

### Sync Strategy Evolution

**Why Hybrid Sync?**:
1. **Pure Polling**: Simple but slow + wasteful (most requests = "no changes")
2. **Pure WebSocket**: Real-time but resource-intensive (millions of open connections)
3. **Hybrid** ✅: WebSocket for active files, polling for stale = optimal

**Fresh vs Stale Classification**:
- Fresh: Edited within last few hours (needs real-time sync)
- Stale: Inactive (can wait minutes for sync)
- Dramatically reduces resource usage while maintaining UX

### Scaling Insights

**Where Bottlenecks Occur**:
- Application servers (solved by direct S3 upload/download)
- Network bandwidth (solved by chunking + parallel transfers)
- Storage costs (solved by deduplication via fingerprinting)
- Latency (solved by CDN caching)

**Fingerprinting Benefits**:
- Content-based addressing (not filename-based)
- Enables deduplication (same file uploaded by multiple users = single S3 object)
- Accurate resume detection (byte-level precision)
- Cryptographic hashes (SHA-256) ensure uniqueness

### Compression Insights

**When to Compress**:
- Compression ratio must exceed compression time cost
- File type matters: text (high ratio), media (low ratio)
- Network speed matters: slow network = compression more valuable
- Client-side decision logic required

**Security + Compression**:
- Always compress BEFORE encrypting
- Encryption randomizes data → makes compression ineffective
- Order: Original → Compress → Encrypt → Upload

### Interview Strategy

**Mid-Level (E4) Expectations**:
- Breadth over depth (80/20)
- Functional high-level design for all requirements
- May not know presigned URLs/chunking initially
- Should reason through problems with guidance: "You're uploading twice, how to avoid?" → eventually reach solution

**Senior (E5) Expectations**:
- More depth (60/40 breadth/depth)
- Quickly establish high-level design, dive deep into large file handling
- Proactively identify trade-offs and optimizations
- Familiar with real-world patterns (multipart upload, presigned URLs)
- Justify architectural decisions with experience

**Common Pitfalls**:
- Uploading file to server before S3 (wastes time + resources)
- Chunking on server instead of client (defeats purpose)
- No progress indicator or resumability for large files
- Ignoring network interruptions
- Not considering CDN for downloads
- Forgetting signed URLs have expiration (security risk)

**What Interviewers Look For**:
- Understanding of presigned URLs (or ability to reason through direct upload)
- Chunking + fingerprinting for large files
- Sync strategies (polling vs WebSocket trade-offs)
- Separation of metadata vs file content storage
- Security considerations (encryption, access control)

## Additional Considerations

- **File Versioning**: Keep multiple versions instead of overwriting (track version number in metadata)
- **Deduplication**: Same file fingerprint → single S3 object (save storage costs)
- **Storage Quotas**: Track user storage usage, enforce limits
- **Virus Scanning**: Scan uploads before making available (out of scope but important in production)
- **Offline Mode**: Queue operations locally, sync when reconnected
- **Conflict Resolution**: Beyond "last write wins" → operational transform or CRDTs for concurrent edits

## Technology Choices

- **Metadata DB**: DynamoDB (NoSQL, horizontal scaling) or Postgres (SQL, ACID guarantees)
- **Blob Storage**: Amazon S3 (scalable, reliable, cheap, presigned URL support)
- **CDN**: CloudFront (supports signed URLs, global edge network)
- **Sync Protocol**: WebSocket for real-time, HTTP polling for periodic checks
- **Compression**: Gzip (universal), Brotli (better ratio), Zstandard (newest, best)
- **Encryption**: HTTPS (transit), S3 SSE (at rest)
