# Email System Design (Gmail/Outlook)

## Overview

Design a large-scale email service supporting sending, receiving, storing, and organizing emails. Modern email services like Gmail handle billions of users and process hundreds of billions of emails daily with features like spam filtering, search, and smart categorization.

**Note**: Email combines real-time messaging (like WhatsApp) with long-term storage (like Google Drive) and complex search functionality (like Google).

## Requirements

### Functional Requirements

**Core Features (Above the Line):**
- ✓ Send emails to one or multiple recipients
- ✓ Receive emails from external and internal senders
- ✓ View inbox and read emails
- ✓ Search emails by sender, subject, content
- ✓ Organize emails (folders, labels, archive)

**Below the Line:**
- Spam detection and filtering
- Email threading/conversations
- Rich text formatting
- Attachments (already covered in send/receive)
- Auto-reply, filters, rules
- Calendar integration
- Contact management

### Non-Functional Requirements

- **High availability**: Email is mission-critical (99.99% uptime)
- **Durability**: Zero data loss (emails must never be lost)
- **Low latency**: Sub-second email delivery for same domain, <30s cross-domain
- **Storage efficiency**: Support large attachments (25MB+) and unlimited storage
- **Scalability**: Support billions of users, trillions of emails
- **Security**: Encryption in transit and at rest, authentication, authorization
- **Compliance**: Data retention policies, GDPR, audit logs

**Key Insight**: Email is unique in requiring both real-time delivery AND long-term durable storage with complex search. Unlike messaging apps that can delete old messages, emails must be retained indefinitely.

## Core Entities

**User**
- userId
- email address
- name
- storage quota
- preferences

**Email**
- emailId
- from (sender)
- to (list of recipients)
- cc, bcc (optional lists)
- subject
- body (plain text + HTML)
- timestamp
- attachments (list of references)
- metadata (headers, size)

**Folder/Label**
- folderId/labelId
- userId
- name
- type (inbox, sent, drafts, custom)

**EmailFolder** (Junction table)
- emailId
- folderId
- userId

**Attachment**
- attachmentId
- filename
- size
- mimeType
- storageLocation (S3 path)

**Key Learning**: Separation of email content from organization (folders/labels) allows flexibility. Gmail uses labels (many-to-many), Outlook uses folders (hierarchical tree).

---

## API Design

### Send Email
```http
POST /emails
Headers: Authorization: Bearer <token>
Body: {
  to: ["user1@domain.com", "user2@domain.com"],
  cc: ["user3@domain.com"],
  bcc: ["user4@domain.com"],
  subject: "Meeting Tomorrow",
  body: "Let's meet at 3pm",
  attachments: ["attachmentId1", "attachmentId2"]
}
Response: {
  emailId: "msg_123456",
  status: "sent",
  timestamp: "2025-11-07T10:30:00Z"
}
```

### Upload Attachment
```http
POST /attachments/presigned_url
Headers: Authorization: Bearer <token>
Body: {
  filename: "report.pdf",
  size: 5242880,
  mimeType: "application/pdf"
}
Response: {
  attachmentId: "att_789",
  presignedUrl: "https://s3.../upload",
  expiresAt: "2025-11-07T11:00:00Z"
}
```

### Get Emails (Inbox)
```http
GET /emails?folder=inbox&limit=50&offset=0
Headers: Authorization: Bearer <token>
Response: {
  emails: [
    {
      emailId: "msg_123",
      from: "sender@domain.com",
      subject: "Hello",
      snippet: "First 100 chars...",
      timestamp: "2025-11-07T09:00:00Z",
      isRead: false,
      hasAttachments: true
    }
  ],
  total: 1523,
  hasMore: true
}
```

### Get Email Details
```http
GET /emails/{emailId}
Headers: Authorization: Bearer <token>
Response: {
  emailId: "msg_123",
  from: "sender@domain.com",
  to: ["recipient@domain.com"],
  subject: "Hello",
  body: "Full email content...",
  bodyHtml: "<html>...</html>",
  timestamp: "2025-11-07T09:00:00Z",
  attachments: [...],
  headers: {...}
}
```

### Search Emails
```http
GET /emails/search?q=from:john subject:meeting&limit=20
Headers: Authorization: Bearer <token>
Response: {
  emails: [...],
  total: 45
}
```

### Organize Emails
```http
PATCH /emails/{emailId}/labels
Headers: Authorization: Bearer <token>
Body: {
  addLabels: ["label_work", "label_important"],
  removeLabels: ["label_inbox"]
}
Response: { status: "success" }
```

**Key Learning**: Use presigned URLs for attachments (same pattern as YouTube/Dropbox). Never send full email bodies in list views (use snippets). userId comes from JWT/session token.

---

## Email Protocol Fundamentals

Before diving into design, understanding email protocols is essential:

### SMTP (Simple Mail Transfer Protocol)
- **Purpose**: Sending emails between mail servers
- **Port**: 25 (unencrypted), 587 (TLS), 465 (SSL)
- **Process**: Client → SMTP server → Recipient's SMTP server
- **Key Commands**: HELO, MAIL FROM, RCPT TO, DATA, QUIT

### IMAP (Internet Message Access Protocol)
- **Purpose**: Accessing/managing emails on server
- **Port**: 143 (unencrypted), 993 (SSL)
- **Features**: Sync across devices, server-side folders, partial fetch
- **Use Case**: Modern email clients (Gmail, Outlook)

### POP3 (Post Office Protocol v3)
- **Purpose**: Download emails to local client
- **Port**: 110 (unencrypted), 995 (SSL)
- **Limitation**: Downloads and deletes from server (old-school)
- **Use Case**: Legacy systems

### Email Structure
```
Headers:
  From: sender@domain.com
  To: recipient@domain.com
  Subject: Test Email
  Date: Thu, 7 Nov 2025 10:30:00 -0800
  Message-ID: <unique-id@domain.com>
  MIME-Version: 1.0
  Content-Type: multipart/mixed; boundary="boundary123"

Body:
  --boundary123
  Content-Type: text/plain; charset="UTF-8"
  
  Plain text version
  
  --boundary123
  Content-Type: text/html; charset="UTF-8"
  
  <html><body>HTML version</body></html>
  
  --boundary123
  Content-Type: application/pdf; name="file.pdf"
  Content-Disposition: attachment; filename="file.pdf"
  
  [binary data]
  --boundary123--
```

---

## High-Level Design

### Architecture Overview

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  Email      │  HTTPS  │   API        │         │   Send      │
│  Client     │────────>│   Gateway    │────────>│   Service   │
│  (Web/App)  │         │   (Auth)     │         │             │
└─────────────┘         └──────────────┘         └─────────────┘
                               │                        │
                               │                        ↓
                               │                  ┌──────────────┐
                               │                  │   SMTP       │
                               │                  │   Gateway    │
                               │                  └──────────────┘
                               │                        │
                               ↓                        ↓
                        ┌─────────────┐         ┌─────────────┐
                        │  Receive    │  SMTP   │  External   │
                        │  Service    │<────────│  Mail       │
                        │             │         │  Servers    │
                        └─────────────┘         └─────────────┘
                               │
                               ↓
                        ┌─────────────┐
                        │  Storage    │
                        │  Service    │
                        └─────────────┘
                               │
                    ┌──────────┴──────────┐
                    ↓                     ↓
             ┌──────────┐          ┌──────────┐
             │  Email   │          │   S3     │
             │  Database│          │ (Attach) │
             └──────────┘          └──────────┘
```

### 1. Send Email

**Flow**:
1. Client uploads attachments via presigned URLs → S3
2. Client calls `POST /emails` with email data + attachmentIds
3. Send Service validates sender, recipients, quota
4. Transaction: Store email in database, create entries in recipients' mailboxes
5. For internal recipients (same domain):
   - Write directly to recipient's inbox
   - Push notification to online clients
6. For external recipients (different domain):
   - Queue to SMTP Gateway
   - SMTP Gateway looks up MX records for recipient domain
   - Establishes SMTP connection, sends email
   - Retries on failure (exponential backoff)

**Components**:

**Send Service**:
- Validates email format, checks spam filters
- Enforces rate limits (e.g., 500 emails/day for free users)
- Deduplicates recipients
- Generates unique Message-ID

**SMTP Gateway**:
- Manages connection pools to external SMTP servers
- Implements retry logic with DLQ (Dead Letter Queue)
- Tracks delivery status (sent, failed, bounced)
- Handles DKIM signing for authentication

**Database Design**:

```
Email Table (Cassandra)
- Partition Key: emailId
- Attributes: from, subject, body, bodyHtml, timestamp, size, headers
- TTL: None (emails stored forever)

EmailRecipient Table (Cassandra)
- Partition Key: userId
- Clustering Key: timestamp (DESC), emailId
- Attributes: folderId, isRead, isStarred, labelIds
- Enables efficient inbox queries: SELECT * WHERE userId = ? ORDER BY timestamp DESC

Attachment Table (Cassandra)
- Partition Key: attachmentId
- Attributes: emailId, filename, size, mimeType, s3Key

SendQueue (Kafka/SQS)
- Messages for external delivery
- Partitioned by recipient domain for batching
```

**Key Learning**: Split email content (rarely changes) from email metadata (frequently updated: read status, labels). Use userId as partition key for mailbox queries to avoid hot partitions.

### 2. Receive Email

**Scenario A: Internal Email (Same Domain)**
- Already handled in Send flow
- Direct database write to recipient mailboxes
- WebSocket/SSE notification to online clients

**Scenario B: External Email (Different Domain)**

**Flow**:
1. External SMTP server looks up MX records for our domain
2. Connects to our SMTP Receiver on port 25/587
3. SMTP Receiver accepts connection, validates:
   - SPF (Sender Policy Framework): Is sending IP authorized?
   - DKIM (DomainKeys Identified Mail): Is signature valid?
   - DMARC (Domain-based Message Authentication): Policy compliance?
4. If valid, parse email (headers, body, attachments)
5. Extract attachments → upload to S3
6. Store email in Email table
7. Create EmailRecipient entries for each recipient
8. Run spam detection (see Deep Dive #1)
9. Push real-time notification to online recipients

**Components**:

**SMTP Receiver**:
- Load balanced cluster of SMTP servers
- Connection pooling (handle thousands of concurrent connections)
- Rate limiting per source IP
- Greylisting for spam prevention

**Email Parser**:
- Parse MIME multipart messages
- Sanitize HTML (remove scripts, dangerous tags)
- Extract attachments
- Virus scanning (ClamAV integration)

**Spam Filter**:
- Bayesian classifier
- Machine learning models
- Blacklist/whitelist checking
- Content analysis (keywords, patterns)

### 3. View Inbox and Read Emails

**Flow (List Inbox)**:
1. Client requests `GET /emails?folder=inbox&limit=50`
2. Storage Service queries EmailRecipient table:
   ```
   SELECT * FROM EmailRecipient 
   WHERE userId = ? AND folderId = 'inbox'
   ORDER BY timestamp DESC
   LIMIT 50
   ```
3. Fetch email metadata (from, subject, snippet) from Email table
4. Return list with snippets (first 100-200 chars)

**Flow (Read Email)**:
1. Client requests `GET /emails/{emailId}`
2. Storage Service fetches full email from Email table
3. Update EmailRecipient: `SET isRead = true`
4. Generate presigned URLs for attachments
5. Return full email with attachment download links

**Optimization - Email Cache**:
- Redis cache for recently accessed emails
- Cache key: `email:{emailId}` → full email JSON
- Cache key: `inbox:{userId}:{page}` → list of email summaries
- TTL: 1 hour for email content, 5 mins for inbox lists
- Invalidate on updates (mark as read, label changes)

**Optimization - Snippet Generation**:
- Pre-compute snippets during send/receive
- Store in EmailRecipient table to avoid fetching full body
- Strip HTML tags, truncate to 150 chars

### 4. Search Emails

**Challenge**: Full-text search across trillions of emails

**Solution: Elasticsearch**

**Flow**:
1. Email ingestion → async index to Elasticsearch
2. Index structure:
   ```json
   {
     "userId": "user123",
     "emailId": "msg_456",
     "from": "john@example.com",
     "to": ["jane@example.com"],
     "subject": "Quarterly Report",
     "body": "The Q3 results show...",
     "timestamp": "2025-11-07T10:00:00Z",
     "labels": ["work", "important"],
     "hasAttachments": true
   }
   ```
3. Client query: `GET /emails/search?q=from:john subject:report`
4. Search Service translates to Elasticsearch query:
   ```json
   {
     "query": {
       "bool": {
         "must": [
           { "term": { "userId": "user123" } },
           { "match": { "from": "john" } },
           { "match": { "subject": "report" } }
         ]
       }
     },
     "sort": [{ "timestamp": "desc" }],
     "size": 20
   }
   ```
5. Elasticsearch returns matching emailIds
6. Fetch email summaries from cache or database

**Search Features**:
- **Basic**: from:, to:, subject:, body:, label:, has:attachment
- **Advanced**: date ranges, boolean operators (AND, OR, NOT)
- **Fuzzy matching**: Typo tolerance
- **Autocomplete**: Suggest recipients, labels

**Indexing Pipeline**:
```
Email Sent/Received → Kafka → Indexer Service → Elasticsearch
                           ↓
                     Database (source of truth)
```

**Key Learning**: Elasticsearch indices partitioned by userId to enable efficient filtering. Reindex jobs handle bulk updates (e.g., label renamed affects thousands of emails).

### 5. Organize Emails (Folders/Labels)

**Gmail Model: Labels (Many-to-Many)**
- An email can have multiple labels
- "Inbox" is just another label
- Archiving = removing "Inbox" label

**Outlook Model: Folders (Hierarchical)**
- An email exists in one folder
- Moving = changing folderId
- Folders can be nested

**We'll implement Labels (more flexible)**:

**Data Model**:
```
Label Table (Cassandra)
- Partition Key: userId
- Clustering Key: labelId
- Attributes: name, color, type (system/custom)

EmailLabel Table (Cassandra)
- Partition Key: userId, emailId
- Clustering Key: labelId
- Enables: "Get all labels for an email"

EmailLabel_ByLabel Table (Cassandra)
- Partition Key: userId, labelId
- Clustering Key: timestamp (DESC), emailId
- Enables: "Get all emails with label X"
```

**Operations**:

**Apply Label**:
1. Insert into EmailLabel and EmailLabel_ByLabel
2. Update Elasticsearch index

**Remove Label**:
1. Delete from both tables
2. Update Elasticsearch index

**Archive** (Remove Inbox label):
1. Remove "inbox" from EmailLabel tables
2. Email still accessible via search, other labels, "All Mail"

**Bulk Operations**:
- Select 100 emails → Apply label
- Use batch writes to Cassandra
- Async update to Elasticsearch

---

## Deep Dives

### Deep Dive #1: Spam Detection

**Multi-Layer Approach**:

**Layer 1: Connection Level (SMTP Receiver)**
- IP reputation (Spamhaus, etc.)
- Rate limiting per IP
- Greylisting (temporarily reject, legitimate servers retry)
- SPF/DKIM/DMARC validation

**Layer 2: Content Analysis**
- Bayesian spam filter (Naive Bayes classifier)
- Train on user's spam/not-spam feedback
- Features: word frequencies, sender reputation, links, attachments

**Layer 3: Machine Learning**
- Deep learning models (LSTM, Transformers)
- Features: subject, body, sender history, link domains
- User behavior: does user typically read emails from this sender?

**Layer 4: User Feedback**
- Mark as spam → train model
- Never spam from this sender → whitelist
- Continuous learning loop

**Spam Score**:
```
if score > 0.9: move to spam
elif score > 0.6: show warning
else: deliver to inbox
```

**Gmail's Approach**:
- TensorFlow models
- Per-user personalization (your spam is different from mine)
- Phishing detection (similar domains, suspicious links)

### Deep Dive #2: Email Threading/Conversations

**Goal**: Group related emails into threads (like Gmail)

**Strategies**:

**Strategy 1: In-Reply-To Header**
- Email headers include `In-Reply-To: <message-id>`
- Also `References: <msg1> <msg2> <msg3>` (full chain)
- Build thread tree from these headers

**Strategy 2: Subject-Based Heuristics**
- Strip "Re:", "Fwd:", "RE:", etc.
- Normalize subject: "Meeting Tomorrow" = "Re: Meeting Tomorrow"
- Group by normalized subject + participants

**Strategy 3: Machine Learning**
- Analyze content similarity
- Time proximity (replies usually within hours/days)
- Participant overlap

**Implementation**:

```
Thread Table (Cassandra)
- Partition Key: threadId
- Attributes: subject, participants, lastUpdated

EmailThread Table (Cassandra)
- Partition Key: emailId
- Attributes: threadId, position
```

**Flow**:
1. New email received
2. Check `In-Reply-To` header → find parent emailId
3. Lookup parent's threadId
4. Add new email to existing thread
5. If no thread found, check subject-based matching
6. Otherwise, create new thread

**UI Display**:
- Show thread summary in inbox (most recent email)
- Click to expand full conversation
- Unread count per thread

### Deep Dive #3: Scalability and Partitioning

**Scale Numbers**:
- 2 billion users
- 300 billion emails sent per day
- 100 TB of new data daily

**Database Partitioning**:

**EmailRecipient Table** (Hot path - inbox queries):
- Partition by userId (each user's mailbox is a partition)
- Clustering by timestamp DESC (newest first)
- Problem: Power users with millions of emails
- Solution: Composite partition key `(userId, year_month)` for very large mailboxes

**Email Table** (Cold storage):
- Partition by emailId
- emailId format: `{timestamp}_{userId}_{random}` for time-based distribution
- Older emails can be moved to cheaper storage (Cassandra → S3 Glacier)

**Attachment Storage**:
- S3 Standard for recent attachments (< 30 days)
- S3 IA (Infrequent Access) for 30-365 days
- S3 Glacier for > 1 year
- Lifecycle policies automate transitions

**SMTP Gateway Scaling**:
- Separate clusters for sending vs receiving
- Send: Partition by recipient domain (batch emails to same domain)
- Receive: Round-robin across receivers, no state
- Connection pooling: Reuse SMTP connections to same domain

**Elasticsearch Scaling**:
- Shard by userId (each user's emails in dedicated shards)
- Hot-warm-cold architecture:
  - Hot nodes: Recent emails (SSDs, high memory)
  - Warm nodes: 30-365 days (cheaper SSDs)
  - Cold nodes: > 1 year (HDDs, compressed)

### Deep Dive #4: Real-Time Notifications

**Goal**: New email arrives → user sees it instantly

**Technologies**:

**WebSockets**:
- Persistent bidirectional connection
- Low latency (< 100ms)
- Each user connects to Notification Service

**Server-Sent Events (SSE)**:
- One-way server → client
- Simpler than WebSockets
- Auto-reconnect built-in

**Architecture**:

```
┌────────────┐         ┌────────────────┐         ┌──────────┐
│  Client    │  WSS    │  Notification  │  Redis  │  Receive │
│  (Browser) │<───────>│  Service       │  Pub/Sub│  Service │
└────────────┘         └────────────────┘         └──────────┘
                              ↑
                              │ Subscribe to userId channel
                              ↓
                         ┌─────────┐
                         │  Redis  │
                         │  Pub/Sub│
                         └─────────┘
```

**Flow**:
1. User connects → Notification Service assigns to server
2. Server subscribes to Redis channel: `user:{userId}:notifications`
3. Email received → Receive Service publishes to Redis:
   ```
   PUBLISH user:user123:notifications {
     "type": "new_email",
     "emailId": "msg_789",
     "from": "john@example.com",
     "subject": "Hello"
   }
   ```
4. Notification Service receives message, pushes to WebSocket
5. Client updates UI in real-time

**Scaling Considerations**:
- 2 billion users, 10% online = 200 million concurrent connections
- Each server handles ~50K connections = 4,000 servers
- Redis Cluster for pub/sub (partition by userId)
- Use consistent hashing for server assignment

**Fallback**:
- WebSocket closed? → Client polls `/emails?since=lastSyncTime`
- Mobile push notifications (FCM, APNs) for offline users

### Deep Dive #5: Data Durability and Disaster Recovery

**Zero Data Loss Guarantee**:

**Write Path**:
1. Email received → Write to Kafka (replication factor = 3)
2. Kafka → Consumer writes to Cassandra
3. Cassandra replication factor = 3 (across availability zones)
4. Async backup to S3 (daily snapshots)

**Multi-Region Replication**:
- Primary region: US-East (writes go here)
- Secondary region: US-West (async replication)
- Tertiary region: EU (async replication)
- RPO (Recovery Point Objective): < 1 minute
- RTO (Recovery Time Objective): < 30 minutes

**Backup Strategy**:
- Cassandra snapshots every 6 hours → S3
- Kafka log retention: 7 days
- S3 versioning enabled (accidental deletes recoverable)
- Cross-region S3 replication

**User-Triggered Recovery**:
- "Undo Send" (within 30 seconds): Email in send queue, cancel delivery
- "Trash" folder: Emails deleted but recoverable for 30 days
- "Archive": Hidden from inbox but searchable

### Deep Dive #6: Security and Compliance

**Encryption**:

**In Transit**:
- TLS 1.3 for all client connections
- SMTP TLS for server-to-server (opportunistic or enforced)
- Certificate pinning for mobile apps

**At Rest**:
- S3 server-side encryption (SSE-S3 or SSE-KMS)
- Cassandra encryption (transparent data encryption)
- Attachment scanning before storage

**Authentication**:
- OAuth 2.0 for third-party apps
- Two-factor authentication (TOTP, SMS, hardware keys)
- Session management (refresh tokens, device tracking)

**Authorization**:
- JWT tokens with userId claim
- Service-to-service: mTLS + service accounts
- Rate limiting per user, per IP

**Compliance**:

**GDPR**:
- Right to access: Export all user data
- Right to deletion: Hard delete user + all emails
- Data residency: EU users' data stored in EU region

**HIPAA** (for healthcare customers):
- Audit logs for all email access
- Encryption key management
- Business Associate Agreement

**Audit Logs**:
```
AuditLog Table (Cassandra)
- Partition Key: userId, date
- Clustering Key: timestamp
- Attributes: action (send, read, delete), emailId, ipAddress, userAgent
```

---

## Capacity Estimation

### User Base
- Total users: 2 billion
- Daily active users: 600 million (30%)
- Emails sent per day: 300 billion
- Emails per active user per day: 500 (send 10, receive 490)

### Storage

**Email Content**:
- Average email size: 50 KB (including headers, formatting)
- Daily new emails: 300 billion × 50 KB = 15 PB/day
- Yearly: 15 PB × 365 = 5.5 EB/year

**Attachments**:
- 20% of emails have attachments
- Average attachment size: 2 MB
- Daily attachments: 300B × 0.2 × 2 MB = 120 PB/day

**Total Daily Storage**: ~135 PB

**Optimization**:
- Deduplication (same attachment sent to multiple recipients)
- Compression (gzip for text, images already compressed)
- Tiered storage (hot/warm/cold)
- Realistic after optimization: ~50 PB/day

### Compute

**API Requests**:
- 600M DAU × 500 emails × 3 requests (list, read, search) = 900 billion requests/day
- QPS: 900B / 86400 = 10.4 million QPS
- Peak (3x average): 31 million QPS

**Application Servers**:
- Each server: 10K QPS
- Required: 31M / 10K = 3,100 servers (for peak)
- With redundancy: ~5,000 servers

**Database**:
- Cassandra cluster: 1,000 nodes (each handles 10M ops/day)
- Elasticsearch cluster: 500 nodes (100 TB index size, 200 GB per node)

### Network

**Bandwidth**:
- 300B emails/day × 50 KB = 15 PB/day
- 15 PB / 86400 seconds = 174 GB/s
- Peak (3x): 520 GB/s = 4.16 Tbps

**SMTP Traffic**:
- 50% external emails: 150B/day
- Average SMTP overhead: 2x (handshake, retry)
- External bandwidth: 300B × 50 KB = 15 PB/day

### Cost Estimation (Rough AWS Pricing)

**Storage** (S3, 50 PB/day):
- S3 Standard (30 days): 1.5 EB × $0.023/GB = $35M/month
- S3 IA (1 year): 18 EB × $0.0125/GB = $225M/month
- S3 Glacier (historical): 100 EB × $0.004/GB = $400M/month
- **Total Storage**: ~$660M/month

**Compute** (EC2, 5000 servers):
- c6i.4xlarge (16 vCPU, 32 GB): $0.68/hour
- 5000 × $0.68 × 730 = $2.5M/month

**Database**:
- Cassandra (1000 nodes, i3.4xlarge): 1000 × $1.248 × 730 = $911K/month
- Elasticsearch (500 nodes): 500 × $1.248 × 730 = $456K/month

**Bandwidth**:
- 15 PB/month × $0.09/GB = $1.35M/month

**Total Monthly Cost**: ~$665M (mostly storage!)

**Key Insight**: Storage dominates costs. Gmail offers "unlimited" storage but likely uses aggressive deduplication and compression.

---

## Trade-offs and Alternatives

### Database Choices

**Cassandra (Chosen)**:
- ✅ Horizontal scalability
- ✅ High write throughput
- ✅ Tunable consistency
- ❌ No complex queries (must design tables per query pattern)

**MySQL + Sharding**:
- ✅ Familiar SQL, transactions
- ✅ Complex queries
- ❌ Sharding complexity, limited scale
- **Use case**: Smaller email services (< 10M users)

**DynamoDB**:
- ✅ Fully managed, auto-scaling
- ✅ Low latency
- ❌ Cost at scale (expensive for 5 EB)
- **Use case**: Startups, AWS-native

### Search Solutions

**Elasticsearch (Chosen)**:
- ✅ Full-text search, fast
- ✅ Mature ecosystem
- ❌ Operational complexity, cost

**Solr**:
- ✅ Similar to Elasticsearch
- ❌ Less popular, smaller community

**Database Full-Text Search** (PostgreSQL, MySQL):
- ✅ Simpler architecture
- ❌ Limited scale, slower
- **Use case**: < 1M users

### Real-Time Notifications

**WebSockets (Chosen)**:
- ✅ True real-time, bidirectional
- ❌ Connection overhead, scaling complexity

**Polling**:
- ✅ Simple, no persistent connections
- ❌ High latency, wasteful (most polls return empty)

**Long Polling**:
- ✅ Lower latency than polling
- ❌ Still wasteful, scaling issues

**Server-Sent Events (SSE)**:
- ✅ Simpler than WebSockets, auto-reconnect
- ❌ One-way only (sufficient for notifications)
- **Alternative**: Use SSE instead of WebSockets

### Attachment Storage

**S3 (Chosen)**:
- ✅ Durable (11 9's), scalable, lifecycle policies
- ✅ Presigned URLs for direct upload/download
- ❌ Cost at massive scale

**Distributed File System** (HDFS, Ceph):
- ✅ Lower cost (self-hosted)
- ❌ Operational burden
- **Use case**: Very large scale (Google-sized)

**Database Blob Storage**:
- ✅ Simple, co-located with metadata
- ❌ Expensive, not designed for large blobs
- **Never use for attachments > 1 MB**

---

## Potential Bottlenecks and Optimizations

### Bottleneck #1: Inbox Query Performance

**Problem**: User with 1 million emails, query slows down

**Solution**:
1. Pagination (limit 50-100 per page)
2. Composite partition key: `(userId, year_month)`
3. Cache recent inbox pages in Redis
4. Lazy load older emails (infinite scroll)

### Bottleneck #2: Search Latency

**Problem**: Searching trillions of emails takes seconds

**Solution**:
1. Shard Elasticsearch by userId (parallel search)
2. Index optimization (merge segments, remove deleted docs)
3. Cache popular searches
4. Limit search to recent emails by default (< 1 year), option to search all

### Bottleneck #3: SMTP Gateway Throughput

**Problem**: 300B emails/day = 3.5M emails/second at peak

**Solution**:
1. Connection pooling (reuse TCP connections to same domain)
2. Batch emails to same domain
3. Horizontal scaling (1000+ SMTP sender instances)
4. Circuit breaker (if domain's server is down, pause sends)

### Bottleneck #4: Attachment Upload

**Problem**: Uploading 100 MB attachment is slow, error-prone

**Solution**:
1. Presigned URLs (upload directly to S3)
2. Multipart upload (parallelize chunks)
3. Resume capability (retry failed chunks)
4. Client-side compression
5. Show progress bar, cancel option

### Bottleneck #5: Database Hot Partitions

**Problem**: Celebrity user gets 10M emails, partition overloaded

**Solution**:
1. Further partition by time: `(userId, year_month, day)`
2. Read replicas for hot partitions
3. Cache aggressively
4. Rate limit senders (anti-spam)

---

## Monitoring and Observability

### Key Metrics

**Availability**:
- Uptime (target: 99.99% = 52 mins downtime/year)
- Error rate (4xx, 5xx responses)

**Latency**:
- API response times (p50, p95, p99)
- Email delivery time (send → receive)
- Search query latency

**Throughput**:
- Emails sent/received per second
- API requests per second
- Database ops per second

**Storage**:
- Disk usage per user
- Attachment storage growth
- Database size

**Business Metrics**:
- Active users (DAU, MAU)
- Emails per user
- Spam catch rate
- User retention

### Alerting

**Critical Alerts** (Page on-call):
- Email delivery failure > 1%
- Database replication lag > 5 mins
- API error rate > 0.5%
- Service down in any region

**Warning Alerts** (Email team):
- Disk usage > 80%
- SMTP send queue > 1 hour backlog
- Search latency > 2 seconds (p95)

### Logging

**Structured Logs** (JSON):
```json
{
  "timestamp": "2025-11-07T10:30:00Z",
  "level": "INFO",
  "service": "send-service",
  "userId": "user123",
  "emailId": "msg_456",
  "action": "send_email",
  "recipients": 3,
  "latency_ms": 125,
  "status": "success"
}
```

**Centralized Logging**:
- All services → Kafka → Elasticsearch (ELK stack)
- Retention: 30 days hot, 1 year cold

**Tracing**:
- Distributed tracing (Jaeger, Zipkin)
- Trace email journey: Client → API → Send Service → SMTP → Receive Service → Storage
- Identify bottlenecks

---

## Summary

### Key Decisions

1. **Database**: Cassandra for scalability, partitioned by userId
2. **Search**: Elasticsearch for full-text search
3. **Attachments**: S3 with presigned URLs
4. **Real-time**: WebSockets/SSE + Redis Pub/Sub
5. **Spam**: Multi-layer (IP, content, ML)
6. **Threading**: Header-based + subject heuristics
7. **Storage Tiering**: Hot (S3 Standard) → Warm (IA) → Cold (Glacier)

### Design Principles

- **Durability First**: Email must never be lost (3x replication, backups)
- **Partition Everything**: By userId to avoid hot spots
- **Cache Aggressively**: Inbox, email content, search results
- **Async Where Possible**: SMTP sending, Elasticsearch indexing, notifications
- **Fail Gracefully**: Retry SMTP sends, queue offline notifications, degrade search

### What We Didn't Cover

- Email aliases (user@domain, user+tag@domain)
- Email forwarding rules
- Auto-responders (vacation mode)
- Email recall (unsend)
- Encryption (PGP, S/MIME)
- Mobile app optimizations (push notifications, background sync)
- A/B testing framework
- Machine learning for smart compose, priority inbox

**Final Thought**: Email is deceptively complex. "Send and receive messages" hides massive challenges in scale, durability, search, and spam. Modern email services are among the most sophisticated distributed systems ever built.
