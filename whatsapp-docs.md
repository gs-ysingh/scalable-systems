# WhatsApp System Design

## Overview

Messaging service for encrypted messages and calls. Originally built on Erlang, renowned for handling high scale with limited resources.

## Requirements

### Functional (In-Scope)

1. Group chats with multiple participants (limit 100)
2. Send/receive messages in real-time
3. Receive offline messages (up to 30 days)
4. Send/receive media attachments

### Functional (Out-of-Scope)

- Audio/Video calling
- Business interactions
- Registration/profile management

### Non-Functional

- Low latency delivery (<500ms)
- Guaranteed message deliverability
- Billions of users support
- Minimal message storage on servers
- Resilience against component failures

## Design Approach

- Treat 1:1 messages as special case of group chats (2 participants)
- Start simple, optimize incrementally
- Address scaling in deep dives

## Core Entities

- **Users**: Individual users of the system
- **Chats**: Conversations with 2-100 participants
- **Messages**: Text and media content
- **Clients**: Multiple devices per user (phone, laptop, tablet)

## API Design

### Why WebSockets?

High-frequency bidirectional updates make WebSockets ideal (REST would be inefficient). Users maintain persistent connections to send/receive commands.

**Key Pattern**: Real-time updates via persistent connections, pub/sub for scaling, careful state management for reliability.

### Client → Server Commands

```javascript
// createChat
{ "participants": [], "name": "" } → { "chatId": "" }

// sendMessage
{ "chatId": "", "message": "", "attachments": [] } → "SUCCESS" | "FAILURE"

// createAttachment
{ "body": ..., "hash": "" } → { "attachmentId": "" }

// modifyChatParticipants
{ "chatId": "", "userId": "", "operation": "ADD" | "REMOVE" } → "SUCCESS" | "FAILURE"
```

### Server → Client Commands

```javascript
// chatUpdate
{ "chatId": "", "participants": [] } → "RECEIVED"

// newMessage
{ "chatId": "", "userId": "", "message": "", "attachments": [] } → "RECEIVED"
```

**Critical**: Clients must ACK received messages so server knows delivery succeeded.

## High-Level Design

### 1. Create Group Chats

**Components**:

- L4 Load Balancer (WebSocket support)
- Chat Service
- DynamoDB tables: Chat, ChatParticipant

**Flow**:

1. User sends `createChat` via WebSocket
2. Service creates Chat + ChatParticipant records (transaction)
3. Returns chatId

**Data Model**:

- **Chat**: Primary key = chatId
- **ChatParticipant**: Composite key (chatId, participantId) + GSI (participantId, chatId) for reverse lookups

### 2. Send/Receive Messages (Single Host, Simple)

**Assumptions** (to be fixed later):

- Single Chat Server
- All users online
- HashMap tracks userId → WebSocket connection

**Flow**:

1. User sends `sendMessage`
2. Server looks up participants from ChatParticipant table
3. Server finds WebSocket connections in HashMap
4. Sends message to all connected participants

### 3. Offline Message Delivery

**New Tables**: Message, Inbox

**Flow (Send)**:

1. User sends `sendMessage`
2. Server looks up participants
3. Transaction: Write to Message table + create Inbox entry per recipient
4. Return SUCCESS/FAILURE
5. Attempt immediate delivery to online users
6. Online clients ACK → delete from Inbox

**Flow (Reconnect)**:

1. Client connects
2. Server queries Inbox for undelivered message IDs
3. Fetches messages from Message table
4. Delivers via `newMessage`
5. Client ACKs → delete from Inbox

**Cleanup**: Cron job deletes messages >30 days old

### 4. Media Attachments

**Solution**: Separate HTTP service for uploads (bandwidth/storage intensive)

- Upload media via HTTP → get attachmentId
- Include attachmentId in message
- Recipients download via HTTP

## Deep Dives

### Scaling to Billions of Users

**Problem**: Single host can't handle billions (realistically 200M concurrent connections need ~100+ servers at 1-2M connections/host)

**Challenge**: Sender and receivers may connect to different Chat Servers

**Solution Options**:

1. **Pub/Sub Pattern** (Recommended)
   - Chat Servers subscribe to topics (e.g., userId or chatId)
   - When message sent, publish to message broker (Kafka/Redis)
   - All subscribed servers receive and deliver to their connected clients

2. **Service Discovery**
   - Track userId → serverId mapping
   - Route messages directly to correct servers
   - More complex, tighter coupling

3. **Broadcast**
   - Send to all servers, each checks if user connected
   - Wasteful but simple

### Multiple Clients Per User

**Problem**: Users have multiple devices (phone, laptop, tablet). Per-user Inbox insufficient.

**Changes**:

1. Add **Clients table**: Track clients by userId
2. Update **Inbox table**: Per-client instead of per-user
3. **Message delivery**: Send to all client IDs for each user
4. **Pub/Sub**: Subscribe by userId (unchanged)
5. **Limits**: Cap at 3 clients/user to prevent storage explosion
6. **Client deactivation**: Remove inactive clients

**Flow**:

1. Look up participants
2. For each participant, look up all clients
3. Create Inbox entry per client
4. Deliver to all connected clients
5. Each client ACKs independently

## Key Learnings

1. **Start Simple**: Begin with single-host solution, scale incrementally
2. **WebSockets for Real-Time**: Bidirectional persistent connections essential for chat
3. **Inbox Pattern**: Solves offline delivery - write to inbox, delete on ACK
4. **ACK Critical**: Client acknowledgments ensure guaranteed delivery
5. **Pub/Sub for Scale**: Decouples servers, enables horizontal scaling
6. **Separate Media**: Use purpose-built HTTP service for bandwidth-intensive attachments
7. **Client vs User**: Track devices separately for accurate delivery tracking
8. **DynamoDB Design**: Composite keys + GSI for bidirectional lookups (chat→users, user→chats)
9. **Interview Strategy**: Define clear scope, solve requirements incrementally, drive deep dives proactively (senior+)
10. **Real-Time Pattern**: Persistent connections + pub/sub + state management = universal pattern for live updates
