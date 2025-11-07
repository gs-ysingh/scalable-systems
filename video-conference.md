# Video Conference System Design

## Overview

Design a video conferencing platform like Google Meet, Zoom, or Microsoft Teams supporting real-time audio/video communication, screen sharing, and chat. Modern video conferencing must handle millions of concurrent meetings with low latency and high reliability.

**Key Challenge**: Unlike video streaming (YouTube), video conferencing requires ultra-low latency bidirectional communication between multiple participants simultaneously.

## Requirements

### Functional Requirements

**Core Features (Above the Line):**
- ✓ Create/join meeting rooms
- ✓ Real-time audio/video streaming between participants
- ✓ Screen sharing
- ✓ Text chat during meetings

**Below the Line:**
- Recording, breakout rooms, virtual backgrounds, polls, hand raising, meeting scheduling

### Non-Functional Requirements

- **Ultra-low latency**: < 200ms end-to-end (critical for natural conversation)
- **High availability**: 99.9% uptime
- **Scalability**: Support 100K+ concurrent meetings
- **Meeting sizes**: 2-100 participants (enterprise: up to 500)
- **Quality**: Adaptive bitrate based on bandwidth
- **Cross-platform**: Web, mobile, desktop

**Key Insight**: Video conferencing prioritizes latency over quality. A slightly pixelated stream that arrives instantly is better than HD video with 2-second delays.

## Core Entities

**User**
- userId
- name
- email
- profilePicture

**Meeting**
- meetingId
- hostId
- title
- startTime
- endTime
- status (active/ended)
- settings (mute on entry, etc.)

**Participant**
- participantId
- meetingId
- userId
- joinTime
- connectionStatus
- audioEnabled
- videoEnabled

**MediaStream**
- streamId
- participantId
- streamType (audio/video/screen)
- codec
- resolution
- bitrate

**ChatMessage**
- messageId
- meetingId
- senderId
- content
- timestamp

## API Design

### REST APIs (Meeting Management)

```http
POST /meetings
Body: { title, startTime, settings }
Response: { meetingId, joinUrl, hostToken }

GET /meetings/{meetingId}
Response: { meetingId, title, host, participants[], status }

POST /meetings/{meetingId}/join
Body: { userId, displayName }
Response: { participantId, webSocketUrl, iceServers }

DELETE /meetings/{meetingId}
Response: { success }
```

### WebSocket APIs (Real-time Signaling)

WebSocket connection established at `wss://signal.service/meetings/{meetingId}`

**Client → Server Commands:**

```javascript
// Join meeting room
{ "type": "join", "participantId": "", "capabilities": ["audio", "video", "screen"] }

// WebRTC Signaling (SDP offer/answer, ICE candidates)
{ "type": "offer", "to": "participantId", "sdp": {...} }
{ "type": "answer", "to": "participantId", "sdp": {...} }
{ "type": "ice-candidate", "to": "participantId", "candidate": {...} }

// Media control
{ "type": "mute", "mediaType": "audio" }
{ "type": "unmute", "mediaType": "video" }

// Chat message
{ "type": "chat", "message": "Hello everyone" }

// Leave meeting
{ "type": "leave" }
```

**Server → Client Commands:**

```javascript
// Participant joined
{ "type": "participant-joined", "participant": {...} }

// Participant left
{ "type": "participant-left", "participantId": "" }

// Forward WebRTC signals
{ "type": "offer", "from": "participantId", "sdp": {...} }
{ "type": "answer", "from": "participantId", "sdp": {...} }
{ "type": "ice-candidate", "from": "participantId", "candidate": {...} }

// Chat message broadcast
{ "type": "chat", "from": "participantId", "message": "", "timestamp": "" }

// Media state change
{ "type": "media-update", "participantId": "", "audio": true, "video": false }
```

**Key Learning**: WebRTC handles actual media transmission. Signaling server only exchanges connection setup information (SDP, ICE candidates). Never send actual video/audio through WebSocket.

## High-Level Design

### Architecture Overview

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Client A   │     │  Client B   │     │  Client C   │
│  (Browser)  │     │  (Mobile)   │     │  (Desktop)  │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │ HTTPS (REST)      │                   │
       ├───────────────────┼───────────────────┤
       │                   │                   │
       ↓                   ↓                   ↓
┌────────────────────────────────────────────────────┐
│            API Gateway / Load Balancer             │
└────────────────────────────────────────────────────┘
       │                   │                   │
       │                   ↓                   │
       │          ┌─────────────────┐          │
       │          │ Meeting Service │          │
       │          │   (REST API)    │          │
       │          └─────────────────┘          │
       │                   │                   │
       │                   ↓                   │
       │          ┌─────────────────┐          │
       │          │   PostgreSQL    │          │
       │          │  (Meetings DB)  │          │
       │          └─────────────────┘          │
       │                                       │
       │ WebSocket (Signaling)                 │
       ├───────────────────┬───────────────────┤
       │                   │                   │
       ↓                   ↓                   ↓
┌────────────────────────────────────────────────────┐
│         Signaling Servers (WebSocket)              │
│              (Stateful, Sticky)                    │
└────────────────────────────────────────────────────┘
       │                   │                   │
       │                   ↓                   │
       │            ┌──────────┐               │
       │            │  Redis   │               │
       │            │ (Pub/Sub)│               │
       │            └──────────┘               │
       │                                       │
       │ UDP/TCP (Media - WebRTC)              │
       ├───────────────────┬───────────────────┤
       │                   │                   │
       ↓                   ↓                   ↓
┌────────────────────────────────────────────────────┐
│        Media Servers (SFU - Selective              │
│         Forwarding Unit)                           │
└────────────────────────────────────────────────────┘
```

### 1. Create/Join Meeting

**Components**:
- API Gateway
- Meeting Service
- PostgreSQL (Meeting metadata)
- Redis (Session cache)

**Flow (Create Meeting)**:

1. User sends `POST /meetings`
2. Meeting Service generates unique `meetingId`
3. Store meeting metadata in PostgreSQL
4. Return `meetingId` and `joinUrl`

**Flow (Join Meeting)**:

1. User sends `POST /meetings/{meetingId}/join`
2. Meeting Service validates meeting exists and is active
3. Create participant record
4. Return `participantId`, `webSocketUrl`, and TURN/STUN server addresses
5. Client establishes WebSocket connection to Signaling Server

**Data Model (PostgreSQL)**:

```sql
-- Meetings table
meetingId (PK), hostId, title, startTime, endTime, 
status, maxParticipants, settings (JSONB)

-- Participants table
participantId (PK), meetingId (FK), userId, 
joinTime, leaveTime, connectionStatus
```

### 2. Real-Time Audio/Video Streaming

**Problem**: How do we transmit audio/video between participants with minimal latency?

**Options Analysis**:

**Option 1: Mesh Architecture (Peer-to-Peer)**
- Each client connects directly to every other client
- Example: 5 participants = each sends 4 streams, receives 4 streams
- ✅ Lowest latency (direct connections)
- ✅ No server infrastructure for media
- ❌ Bandwidth scales O(n²) - doesn't work beyond ~4 participants
- ❌ Client upload bandwidth becomes bottleneck

**Option 2: MCU (Multipoint Control Unit)**
- Server receives all streams, decodes, mixes into single stream, encodes, sends to all
- ✅ Minimal client bandwidth (1 up, 1 down)
- ❌ High server CPU (decode/encode for each stream)
- ❌ Added latency (decode + encode)
- ❌ Loses individual stream quality control

**Option 3: SFU (Selective Forwarding Unit)** ✅ **CHOSEN**
- Server receives all streams, forwards without processing
- Each client sends 1 stream, receives N-1 streams
- ✅ Balanced bandwidth (reasonable for clients)
- ✅ Low latency (no processing)
- ✅ Low server CPU (just forwarding)
- ✅ Clients can choose quality per stream
- ✅ Scales well with server horizontal scaling

**Key Insight**: SFU is the industry standard (Zoom, Meet, Teams). Trade client download bandwidth for low latency and scalability.

**WebRTC + SFU Flow**:

1. **Participant A joins**:
   - Connects WebSocket to Signaling Server
   - Sends "join" command
   - Receives list of existing participants

2. **For each existing participant**:
   - A creates WebRTC PeerConnection
   - A generates SDP offer (describes A's media capabilities)
   - A sends offer via WebSocket to Signaling Server
   - Server forwards offer to target participant
   - Target responds with SDP answer
   - Both exchange ICE candidates (network paths)
   - WebRTC establishes direct media connection to SFU

3. **Media flows**:
   - A sends single media stream to SFU
   - SFU forwards to each participant
   - A receives media streams from all others

**Components**:

- Signaling Server: WebSocket connections, forwards signaling messages
- SFU Media Server: Receives/forwards RTP packets
- Redis Pub/Sub: Synchronizes state across Signaling Servers

### 3. Screen Sharing

**Implementation**: Same as video streaming but different track type

1. Client captures screen using `getDisplayMedia()` API
2. Creates new WebRTC track with type "screen"
3. Adds track to existing PeerConnection
4. SFU forwards screen stream separately
5. Receivers render in separate UI element

**Optimization**: Screen content often has less motion than video
- Use lower frame rate (5-10 fps vs 30 fps)
- Prefer CPU-optimized codecs for text clarity
- Higher resolution (1080p+) since bandwidth is lower at low FPS

### 4. Text Chat

**Why separate from media?**: Chat requires reliable, ordered delivery (TCP-like). Media prioritizes speed over reliability (UDP-like).

**Implementation**:

1. User sends chat message via WebSocket
2. Signaling Server receives message
3. Store in database for persistence (PostgreSQL)
4. Publish to Redis Pub/Sub channel (meetingId)
5. All Signaling Servers with participants in that meeting subscribe
6. Broadcast to connected participants via WebSocket

**Optimization**: Keep recent messages in Redis cache for quick retrieval when participants join

## Deep Dives

### Deep Dive #1: Scaling Media Servers (SFU)

**Challenge**: Single SFU can handle ~100-200 participants. How to scale?

**Problem Breakdown**:
- 100 participants
- Each sends 1 video (2 Mbps), 1 audio (50 Kbps)
- SFU receives: 100 × 2.05 Mbps = 205 Mbps
- SFU sends: 99 streams to each participant = 99 × 2.05 × 100 = ~20 Gbps
- Single server bandwidth limit reached

**Solution 1: Cascading SFUs**

```
Meeting with 200 participants split across 2 SFUs:

SFU-A (100 participants)  ←→  SFU-B (100 participants)
```

- Each SFU acts as a participant to the other
- Reduces bandwidth per server
- Adds minimal latency (~10-20ms)
- Implementation: Participants randomly assigned to SFUs

**Solution 2: Simulcast**

- Client sends multiple qualities (720p, 360p, 180p) simultaneously
- SFU forwards appropriate quality based on receiver's bandwidth/UI
- Gallery view: send 180p for small tiles
- Active speaker: send 720p for main view
- Reduces downstream bandwidth significantly

**Solution 3: Smart Routing by Region**

- Deploy SFUs in multiple regions
- Route participants to nearest SFU
- Inter-SFU communication for cross-region meetings
- Reduces latency for global meetings

**Combined Approach**: All three techniques together
- Regional SFUs for low latency
- Cascading for large meetings
- Simulcast for bandwidth optimization

### Deep Dive #2: Handling Network Instability

**Problem**: Internet connections are unreliable. How to maintain quality?

**Techniques**:

**1. Adaptive Bitrate**
- Monitor RTT, packet loss, jitter
- Dynamically adjust video resolution/framerate
- Client requests lower quality from SFU during congestion

**2. Forward Error Correction (FEC)**
- Send redundant data packets
- Receiver can reconstruct lost packets
- Increases bandwidth but improves quality

**3. Jitter Buffer**
- Buffer incoming packets briefly
- Smooth out network timing variations
- Trade 50-100ms latency for smoother playback

**4. Packet Loss Concealment**
- Audio: Interpolate missing frames
- Video: Repeat previous frame
- Better than silence/freeze

**5. NACK (Negative Acknowledgment)**
- Receiver requests retransmission of lost packets
- Only works if RTT is low (<100ms)
- WebRTC built-in feature

**Monitoring**:
- Track per-participant metrics: RTT, packet loss, bitrate
- Alert when quality degrades
- Suggest troubleshooting (close other apps, switch to ethernet)

### Deep Dive #3: Signaling Server Architecture

**Challenge**: Maintain WebSocket connections for millions of users

**Requirements**:
- Stateful (maintains WebSocket connections)
- Must route messages between specific participants
- Need horizontal scaling

**Architecture**:

```
┌──────────────┐         ┌──────────────┐
│  Client A    │────────→│ Signaling    │
│              │  WS     │ Server 1     │
└──────────────┘         └──────┬───────┘
                                │
                                ↓
┌──────────────┐         ┌─────────────────┐
│  Client B    │────────→│  Redis Pub/Sub  │
│              │  WS     │   (per meeting) │
└──────────────┘         └──────┬──────────┘
                                │
                                ↓
┌──────────────┐         ┌──────────────┐
│  Client C    │────────→│ Signaling    │
│              │  WS     │ Server 2     │
└──────────────┘         └──────────────┘
```

**Implementation**:

1. **Sticky Load Balancing**: Route reconnections to same server
2. **Meeting-Based Sharding**: All participants in a meeting connect to same server (if possible)
3. **Redis Pub/Sub**: When participants on different servers:
   - Server 1 publishes signal to Redis channel (meetingId)
   - Server 2 subscribes to same channel
   - Receives signal and forwards to connected client

**Scalability**:
- Each server handles 50K WebSocket connections
- For 5M concurrent users → 100 servers
- Auto-scale based on connection count

**State Management**:
- Store "participantId → serverId" mapping in Redis
- When routing signal, lookup target server
- If on different server, use Redis Pub/Sub

### Deep Dive #4: Recording Meetings

**Challenge**: Record audio/video while maintaining low latency for live participants

**Solution**: Dedicated Recording Service

**Architecture**:

```
SFU Media Server
    ↓ (stream copy)
Recording Bot (acts as silent participant)
    ↓
Raw Media Storage (S3)
    ↓
Async Processing Pipeline
    ↓
- Mux audio/video
- Add timestamps
- Generate chapters
- Create highlights
    ↓
Final Recording (S3 + CDN)
```

**Flow**:

1. Host enables recording
2. Meeting Service spawns Recording Bot
3. Bot joins as hidden participant
4. SFU forwards all streams to bot
5. Bot writes raw streams to S3 (fast, no processing)
6. When meeting ends, trigger processing pipeline
7. Combine streams, add overlays (name tags, timestamps)
8. Generate MP4 output
9. Upload to S3, make available via CDN

**Optimization**:
- Record in cloud (not client-side) for reliability
- Use SFU's simulcast to record highest quality stream
- Parallel processing for large meetings (split into segments)

### Deep Dive #5: Security & Privacy

**Threats**:
- Unauthorized meeting access
- Man-in-the-middle attacks
- Eavesdropping on media streams

**Solutions**:

**1. Meeting Access Control**
- Generate unique, hard-to-guess meeting IDs (UUID)
- Optional: Password protection
- Optional: Waiting room (host approves participants)
- Token-based authentication for join

**2. End-to-End Encryption (E2EE)**
- Media encrypted at sender, decrypted at receiver
- SFU cannot decrypt content (only forwards encrypted packets)
- Trade-off: Cannot do server-side recording/transcription
- Implementation: Use Insertable Streams API in WebRTC

**3. Transport Security**
- TLS for REST APIs (HTTPS)
- WSS for WebSocket (TLS)
- DTLS-SRTP for WebRTC media (encrypted UDP)
- Validate SSL certificates

**4. TURN Server Authentication**
- Time-limited credentials for TURN servers
- Prevent unauthorized relay usage
- Rotate credentials per session

## System Characteristics

### Capacity Estimation

**Assumptions**:
- 100K concurrent meetings
- Average 5 participants per meeting
- 500K concurrent users
- Average meeting duration: 30 minutes

**Bandwidth**:
- Per user: 2 Mbps video upload, 8 Mbps download (4 other participants)
- Total upload: 500K × 2 Mbps = 1 Tbps
- Total download: 500K × 8 Mbps = 4 Tbps
- **SFU Infrastructure**: 5 Tbps total throughput

**Storage (Recordings)**:
- 10% of meetings recorded = 10K meetings
- 30 min × 5 participants × 2 Mbps = 900 MB per meeting
- Daily: 10K × 900 MB = 9 TB
- Yearly: 9 TB × 365 = 3.3 PB

**Database**:
- 100K meetings × 1 KB metadata = 100 MB
- 500K participants × 500 bytes = 250 MB
- Daily writes: ~350 MB (easily handled by PostgreSQL)

### Technology Choices

**Meeting Service**: Node.js / Go (async I/O for APIs)
**Signaling Servers**: Node.js (native WebSocket support)
**SFU Media Servers**: C++ (low-level performance) - mediasoup, Janus
**Database**: PostgreSQL (meeting metadata, ACID compliance)
**Cache**: Redis (session cache, pub/sub)
**Storage**: S3 (recordings, media files)
**CDN**: CloudFront / Akamai (deliver recordings)
**Message Queue**: Kafka (recording pipeline, analytics)

### Monitoring & Observability

**Key Metrics**:

- **Quality Metrics**: Average RTT, packet loss %, jitter, bitrate
- **Availability**: Meeting service uptime, SFU uptime
- **Usage**: Concurrent meetings, participants per meeting, duration
- **Performance**: WebSocket connection time, ICE gathering time, time-to-first-frame

**Alerting**:
- Packet loss > 5% → investigate network issues
- Average RTT > 300ms → check SFU routing
- Meeting join failures > 1% → check signaling servers
- SFU CPU > 80% → scale horizontally

**User Feedback**:
- Post-meeting quality survey
- Connection quality indicator in UI
- Diagnostic logs collection (with permission)

## Trade-offs & Alternatives

### SFU vs MCU vs Mesh

- **Chose SFU**: Best balance of latency, quality, and scalability
- **MCU**: Better for very large meetings (webinars) where most are view-only
- **Mesh**: Only for small meetings (2-4 people)

### WebRTC vs Custom Protocol

- **Chose WebRTC**: Browser native, battle-tested, handles NAT traversal
- **Custom**: More control but requires client installation, harder NAT traversal

### Monolithic vs Microservices

- **Chose Microservices**: Separate meeting management, signaling, media services
- **Reasoning**: Independent scaling, fault isolation, technology flexibility
- **Monolithic**: Simpler for small scale, but limits scaling patterns

### Regional vs Global SFUs

- **Chose Regional**: Lower latency for most meetings
- **Cross-region when needed**: Automatic for international meetings
- **Trade-off**: More infrastructure cost but better UX

## Future Enhancements

1. **AI Features**:
   - Live transcription / closed captions
   - Real-time translation
   - Noise suppression / echo cancellation
   - Virtual backgrounds

2. **Advanced Collaboration**:
   - Whiteboarding
   - Collaborative document editing
   - Polls and Q&A
   - Breakout rooms

3. **Analytics**:
   - Engagement metrics (attention tracking)
   - Speech analytics (talk time per participant)
   - Meeting insights and summaries

4. **Mobile Optimization**:
   - Lower bitrate codecs for cellular
   - Battery optimization
   - Background mode support

## References

- WebRTC Specification: https://webrtc.org/
- SFU Architecture: mediasoup documentation
- Zoom's Engineering Blog: https://medium.com/zoom-developer-blog
- RFC 8829 (JSEP - JavaScript Session Establishment Protocol)
