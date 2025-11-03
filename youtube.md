# YouTube System Design

## Overview

Design a video-sharing platform supporting video upload, streaming, and playback. YouTube is the second most visited website in the world, handling ~1M video uploads and 100M video views per day.

**Note**: Similar to Dropbox design for large file handling. Focus on video-specific aspects like streaming and adaptive bitrate.

## Requirements

### Functional

- Upload videos
- Watch (stream) videos

### Non-Functional

- High availability (prioritize availability over consistency)
- Support large videos (10s of GBs)
- Low latency streaming (even in low bandwidth environments)
- Scale to 1M uploads/day, 100M views/day
- Resumable uploads

**Key Insight**: Non-functional requirements define the complexity. "Upload" and "watch" sound simple but require sophisticated solutions for scale and performance.

## Core Entities

- **User**: Uploader or viewer
- **Video**: Raw video content (frames, audio)
- **VideoMetadata**: Uploader info, transcript URLs, manifest file URLs, upload status

## API Design

```http
POST /presigned_url { VideoMetadata } -> { presignedUrl }
GET /videos/{videoId} -> VideoMetadata
```

**Note**: APIs evolve during design. Initial `POST /upload` becomes `POST /presigned_url` for direct S3 upload. VideoMetadata contains URLs to video content, not the content itself.
## Video Streaming Fundamentals

Before diving into the design, understanding basic video concepts is essential:

**Video Codec**: Compresses/decompresses video for efficient storage and transmission
- Examples: H.264, H.265 (HEVC), VP9, AV1, MPEG-2, MPEG-4
- Trade-offs: Compression time, platform support, compression ratio, quality (lossy vs lossless)

**Video Container**: File format storing video data, audio, and metadata
- Different from codec: codec = how video is compressed, container = how it's stored
- Support varies by device/OS

**Bitrate**: Bits transmitted per second (kbps or mbps)
- High resolution + high FPS = higher bitrate (more data to transfer)
- Compression affects bitrate (efficient codecs = smaller size)

**Manifest Files**: Text documents describing video streams
- **Primary Manifest**: Lists all available video versions (different formats)
- **Media Manifests**: List segments for each version (each segment = few seconds)
- Video players use these as "index" to stream segments

**Video Format**: Shorthand for container + codec combination

## High-Level Design

### 1. Upload Videos

**Storage Decisions**:

**Video Metadata** → Cassandra:
- Scale: 1M uploads/day = 365M records/year
- Needs horizontal partitioning
- Partition by `videoId` (simple point lookups, no bulk access needed)
- High availability via leaderless replication

**Video Data** → S3:
- Use presigned URLs for direct upload (same as Dropbox pattern)
- Multi-part upload for large files
- Bypasses application servers

**Pattern: Handling Large Blobs**

Multi-GB video files uploaded directly to S3:
1. Client requests presigned URL from Video Service
2. Video Service generates signed URL, saves VideoMetadata with "uploading" status
3. Client uploads directly to S3 using multipart upload
4. S3 event notification → Lambda → updates VideoMetadata to "uploaded"

**Benefits**: No double upload, saves server resources, faster uploads

**What to Store?**

Options for video storage:
- Original format only (simple but poor streaming experience)
- ✅ **Multiple formats**: Different codecs/containers + segment files + manifest files
  - Enables adaptive bitrate streaming
  - Requires post-processing pipeline (see Deep Dive #1)

### 2. Watch Videos

**Flow**:
1. Client requests `GET /videos/{videoId}`
2. Video Service returns VideoMetadata (contains manifest file URLs)
3. Client fetches primary manifest file
4. Client selects appropriate format based on device/network
5. Client fetches media manifest for selected format
6. Client streams video segments sequentially

**Streaming Strategy**:
- Store segments in different formats in S3
- Manifest files guide video player on which segments to fetch
- Client adapts quality based on network conditions (adaptive bitrate streaming)

## Deep Dives

### 1. Video Processing for Adaptive Bitrate Streaming

**Goal**: Support smooth playback by allowing clients to download video segments in varying formats based on network conditions.

**Pipeline Output**:
- Video segment files in different formats (codec + container combinations) → S3
- Manifest files (primary + media manifests) → S3
- Media manifests reference segment files

**Processing Steps** (Directed Acyclic Graph - DAG):

1. **Split**: Divide original video into segments (using ffmpeg)
2. **Transcode**: Convert each segment to different codecs/containers (parallel processing)
   - Most CPU-intensive operation
   - Process on multiple worker nodes/cores simultaneously
3. **Process**: Audio extraction, transcript generation (parallel per segment)
4. **Generate Manifests**: Create primary + media manifest files
5. **Complete**: Mark upload as "complete" in VideoMetadata

**DAG Characteristics**:
- Parallelizable: Segments have no dependencies, can process concurrently
- Fan-out/fan-in: Multiple workers process different segments
- Orchestration: Use existing tools like **Temporal** to manage workflow

**Implementation Notes**:
- S3 stores temporary data (segments, audio files)
- Workers pass URLs (not files) to avoid data transfer overhead
- Extreme parallelism for transcoding (CPU-bound work)
- Alternative: Client-side segmentation for pipelined processing (upload + process simultaneously)

**Why This Matters**:
- Smooth playback across devices (phones, tablets, desktops)
- Adapts to changing network conditions
- Users don't experience buffering

### 2. Resumable Uploads

**Same as Dropbox pattern** - chunked upload with fingerprinting:

**Implementation**:
1. Client chunks video (5-10MB), calculates SHA-256 fingerprint per chunk
2. POST VideoMetadata with chunks array: `[{ fingerprint, status: "NotUploaded" }]`
3. Upload chunks to S3 via multipart upload
4. S3 event notification → Lambda → update chunk status to "Uploaded"
5. Resume: Fetch VideoMetadata, skip uploaded chunks

**Key Insight**: AWS multipart upload handles this, but understanding the mechanics shows depth in interviews.

### 3. Scaling to 1M Uploads/Day, 100M Views/Day

**Component Analysis**:

**Video Service**:
- Stateless, horizontally scalable
- Load balancer distributes traffic
- Handles presigned URL requests + metadata queries

**Video Metadata (Cassandra)**:
- Scales via consistent hashing + leaderless replication
- Partition by videoId (uniform distribution)
- **Problem**: "Hot" videos create read bottlenecks on nodes

**Video Processing Service**:
- Scales via elastic worker nodes
- Internal queue handles upload bursts
- Queue length triggers auto-scaling

**S3**:
- Multi-region, elastic scaling
- **Problem**: Faraway users experience latency/buffering

**Optimizations**:

**For Hot Videos**:
- **Cassandra Replication**: Tune replication factor (multiple nodes serve same video)
- **Distributed Cache**: LRU cache partitioned by videoId
  - Stores popular video metadata
  - Insulates database from read load
  - Faster retrieval for viral videos

**Pattern: Scaling Reads**

YouTube's extreme read-to-write ratio (millions watch, one upload):
- Aggressive metadata caching
- CDN distribution for video content
- Read replicas / replication for database
- Viral videos = read hotspots requiring special handling

**For Geographic Distribution**:
- **CDN**: Cache video segments + manifest files on edge servers
  - Geographically close to users
  - Reduces latency and buffering
  - CDN-only serving (no backend interaction after initial metadata fetch)

**Final Architecture Components**:
- Video Service (presigned URLs, metadata)
- Cassandra (video metadata, partitioned by videoId)
- Distributed Cache (LRU, popular videos)
- Video Processing Service (DAG orchestration, worker nodes)
- S3 (original + processed videos, segments, manifests)
- CDN (edge caching for popular content)

## Key Learnings & Insights

### Design Patterns

**Pattern: Handling Large Blobs**
- Direct upload to S3 via presigned URLs
- Multipart/chunked uploads for reliability
- Bypasses application servers
- Applies to: Any large media system (photos, podcasts, game assets)

**Pattern: Scaling Reads**
- Read-to-write ratio = millions to one for viral content
- Multi-layer caching: Application cache + CDN
- Database replication for read load distribution
- Geographic distribution via edge servers

### Video-Specific Insights

**Adaptive Bitrate Streaming**:
- Not optional at YouTube scale
- Requires post-processing pipeline (not real-time)
- Segment-based architecture enables quality switching mid-stream
- Manifest files = control plane, segments = data plane

**Processing Pipeline as DAG**:
- Video transcoding is CPU-bound and embarrassingly parallel
- Each segment independent = maximum parallelism
- Orchestration tools (Temporal) manage complex workflows
- Temporary data in S3 (not in-memory or disk on workers)

**Codec/Container Trade-offs**:
- Different devices need different formats
- No single "best" codec (platform support vs compression vs speed)
- Must generate multiple formats per video
- Storage cost vs user experience balance

### Storage Insights

**Why Cassandra?**
- 365M+ records/year needs horizontal partitioning
- Point lookups by videoId (no complex queries)
- High availability via leaderless replication
- Could also use sharded SQL, but Cassandra fits access pattern

**Partitioning Strategy**:
- Partition by videoId (uniform distribution)
- Different from systems needing domain-scoped consistency
- No bulk access requirements = simple partitioning

**Hot Data Problem**:
- Popular videos create "hot" partitions
- Solution: Replication + caching
- Cache hit ratio critical for viral content

### Scaling Insights

**Where Bottlenecks Occur**:
- Database reads for popular videos (solved by cache + replication)
- Geographic latency for video content (solved by CDN)
- Processing queue during upload spikes (solved by elastic scaling)
- Upload bandwidth (solved by direct S3 + multipart)

**CDN Strategy**:
- Cache both segments AND manifest files
- Enables complete CDN-only streaming (no backend calls)
- Edge servers handle 99%+ of traffic for popular videos
- Massive cost savings on bandwidth and compute

**Cache Strategy**:
- Distributed cache partitioned by videoId
- LRU eviction (popular videos stay, old videos evicted)
- Insulates database from read spikes
- Separate from CDN (metadata vs content)

### Interview Strategy

**Common Pitfalls**:
- Not understanding video streaming basics (codec, bitrate, manifest)
- Uploading to server before S3 (double upload)
- Serving video through application servers (bandwidth waste)
- No adaptive bitrate strategy (poor UX in varying networks)
- Ignoring read-heavy nature (no caching strategy)

**What Interviewers Look For**:
- Understanding of video fundamentals (don't need expert-level)
- Presigned URLs for direct S3 upload
- Post-processing pipeline for multiple formats
- Adaptive bitrate streaming concept
- Caching + CDN for read-heavy workload
- DAG-based processing for parallelism

**Deep Dive Selection**:
- Start with functional requirements (upload, watch)
- Use non-functional requirements to guide deep dives
- Video processing is often the "meatiest" deep dive
- Scaling discussion shows system thinking
- Mention resumable uploads (links to Dropbox knowledge)

### Architecture Principles

**Separation of Concerns**:
- Upload path: Client → S3 (direct)
- Processing path: S3 → Workers → S3 (segments)
- Streaming path: Client → CDN → S3 (cache miss)
- Metadata path: Client → Service → Cassandra → Cache

**Asynchronous Processing**:
- Upload and processing are separate phases
- User doesn't wait for transcoding
- Background workers with queue-based scaling
- Status tracking in VideoMetadata

**Read vs Write Optimization**:
- Write path: Direct S3, async processing
- Read path: Multi-layer caching, CDN distribution
- Optimize differently based on access patterns
- YouTube = extreme read optimization

## Additional Considerations

**Optimizations**:
- **Pipelined Upload**: Client segments video, backend processes as chunks arrive (faster time-to-publish)
- **Resume Playback**: Store user progress per video (requires additional state)
- **View Counts**: Exact (strong consistency) vs estimated (eventual consistency) trade-offs

**Out of Scope** (but worth mentioning):
- Search & recommendations (separate indexing system)
- Comments (separate service)
- Content moderation (ML pipelines)
- DRM & copyright detection
- Analytics & monitoring

## Technology Choices

- **Metadata DB**: Cassandra (horizontal partitioning, high availability)
- **Blob Storage**: Amazon S3 (scalable, multi-region, presigned URLs)
- **Cache**: Distributed cache (Redis/Memcached) with LRU eviction
- **CDN**: CloudFront or similar (global edge network)
- **Processing**: Temporal for orchestration, ffmpeg for transcoding
- **Workers**: Elastic compute (EC2, Kubernetes) for parallel processing
- **Queue**: Internal to processing service (SQS, Kafka, or custom)

