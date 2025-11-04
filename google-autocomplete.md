# Google Autocomplete System Design

## Overview

Design a typeahead/autocomplete system that provides real-time search suggestions as users type. Similar to Google Search's autocomplete feature, it should predict and suggest complete queries based on partial input, search trends, and popularity.

**Scale**: Handle billions of searches/day, millions of concurrent users, sub-100ms response time.

## Requirements

### Functional

- Provide top N suggestions (typically 5-10) as user types
- Personalized suggestions based on user search history
- Support trending/popular queries
- Handle prefix matching (user types "face" → suggest "facebook", "face masks", etc.)

### Non-Functional

- **Low latency**: < 100ms response time (ideally < 50ms)
- **High availability**: 99.99% uptime
- **Scalability**: Billions of searches/day, millions of concurrent users
- **Consistency**: Eventual consistency acceptable (trending queries don't need instant global sync)
- **Fault tolerance**: Graceful degradation if components fail

### Out of Scope

- Full text search (different problem)
- Spell correction (mention as extension)
- Image/video search suggestions
- Voice input

## Core Entities

- **User**: Person performing search
- **Query**: Search string (prefix during typing, complete when submitted)
- **Suggestion**: Predicted query completion with metadata (popularity, timestamp)
- **SearchLog**: Historical search data for analytics

## API Design

```http
GET /suggestions?prefix={prefix}&limit={N}
  Headers: Authorization: Bearer {token}
  -> { suggestions: [{ text, score, type }] }

POST /search { query, timestamp }
  Headers: Authorization: Bearer {token}
  -> { success: true }
```

**Note**: `userId` is extracted from the authenticated session/JWT token, not passed as a parameter. This is more secure and follows REST best practices.

**Response Example**:
```json
{
  "suggestions": [
    { "text": "facebook login", "score": 95000, "type": "trending" },
    { "text": "facebook messenger", "score": 87000, "type": "popular" },
    { "text": "facebook marketplace", "score": 72000, "type": "popular" },
    { "text": "facebook stock", "score": 45000, "type": "recent" },
    { "text": "facebook careers", "score": 38000, "type": "personalized" }
  ]
}
```

## High-Level Design

### Simple Approach (Initial Version)

```
┌──────────┐
│  Client  │ (Types search query)
└────┬─────┘
     │
     ▼
┌─────────────────┐
│ Load Balancer   │
└────┬────────────┘
     │
     ├─────────────────────────────────────┐
     │                                     │
     ▼                                     ▼
┌──────────────────┐              ┌──────────────────┐
│ Query Service    │              │ Data Gathering   │
│                  │              │    Service       │
│ • Receives prefix│              │                  │
│ • Queries DB     │              │ • Logs searches  │
│   using LIKE     │              │ • Updates freq   │
│                  │              │                  │
└────┬─────────────┘              └────┬─────────────┘
     │                                 │
     ▼                                 ▼
┌─────────────────────────────────────────┐
│           Database (MySQL/Postgres)     │
│                                         │
│  Table: search_queries                  │
│  ┌──────────┬───────────┬────────────┐ │
│  │  query   │ frequency │ last_update│ │
│  ├──────────┼───────────┼────────────┤ │
│  │ facebook │  95000    │ 2025-11-04 │ │
│  │ face ma..│  45000    │ 2025-11-04 │ │
│  │ factory  │  12000    │ 2025-11-04 │ │
│  └──────────┴───────────┴────────────┘ │
│                                         │
│  Query: SELECT query, frequency         │
│         FROM search_queries             │
│         WHERE query LIKE 'fac%'         │
│         ORDER BY frequency DESC         │
│         LIMIT 10                        │
└─────────────────────────────────────────┘
```

**Limitations of LIKE Query Approach:**
- ❌ Slow for large datasets (millions of queries)
- ❌ No index optimization for prefix matching
- ❌ Full table scan required
- ❌ Cannot handle billions of searches/day
- ❌ High latency (100ms+)

**Alternative Simple Approach: In-Memory Trie**

Instead of using database LIKE queries, you can also use a Trie data structure directly:

```
┌──────────────────┐              ┌──────────────────┐
│ Query Service    │              │ Data Gathering   │
│                  │              │    Service       │
│ • Receives prefix│              │                  │
│ • Queries Trie   │              │ • Logs searches  │
│   in-memory      │              │ • Updates DB     │
│                  │              │                  │
└────┬─────────────┘              └────┬─────────────┘
     │                                 │
     ▼                                 │
┌─────────────┐                        │
│ Trie Cache  │                        │
│ (In-Memory) │                        │
│             │◄───────────────────────┤
│ • Fast O(k) │         Periodic       │
│   lookup    │         Rebuild        │
│ • No DB hit │      (hourly/daily)    │
└─────────────┘                        │
                                       ▼
                              ┌─────────────────┐
                              │    Database     │
                              │ (query + freq)  │
                              └─────────────────┘
```

**Key Insight: Trie doesn't need real-time updates**
- ✅ Build Trie from database periodically (hourly or daily)
- ✅ Serve all queries from in-memory Trie (fast lookups)
- ✅ Log searches to database asynchronously
- ✅ Rebuild Trie on schedule (not on every search)
- ✅ Much faster than LIKE queries (O(k) vs O(n))

**Trade-offs:**
- Freshness: New trending queries appear after rebuild (acceptable delay)
- Memory: Entire Trie must fit in server memory
- Simple to implement for small-to-medium scale

**This approach works for:**
- Small scale (< 100k unique queries)
- Low traffic (< 1k requests/second)
- MVP or prototype phase

### Scalable Approach (Production Version)

```
┌──────────┐
│  Client  │ (Types search query)
└────┬─────┘
     │
     ▼
┌─────────────────┐      ┌──────────────┐      ┌─────────────┐
│ Load Balancer   │─────▶│   Search     │─────▶│   Kafka     │
│                 │      │   Service    │      │  (Events)   │
└─────────────────┘      └──────────────┘      └──────┬──────┘
                                                       │
                         ┌─────────────────────────────┤
                         │                             │
                         ▼                             ▼
                  ┌─────────────┐            ┌─────────────────┐
                  │  Analytics  │            │   Data Lake     │
                  │  Service    │───────────▶│  (S3/Hadoop)    │
                  └─────────────┘            └────────┬────────┘
                                                      │
                                                      │ (Reads aggregated data)
                                                      │
                                               ┌──────┴──────┐
                                               │Trie Builder │
                                               │  Service    │
                                               └──────┬──────┘
                                                      │
                                                      ▼
                                               ┌─────────────┐
                                               │Redis Cluster│
                                               │   (Trie)    │
                                               └─────────────┘
```

### 1. Collecting Search Data

**Components**:
- API Gateway / Load Balancer
- Search Service (stateless, horizontally scalable)
- Kafka (message queue for search events)
- Analytics Service (processes search logs)
- Data Lake (Hadoop/S3 for historical data)

**Flow**:
1. User submits complete search query
2. Search Service logs to Kafka topic
3. Kafka persists search event: `{ userId, query, timestamp, location, device }`
4. Analytics Service consumes from Kafka
5. Aggregate data written to Data Lake
6. Trie Builder reads from Data Lake and updates Redis

**Diagram**:
```
User Search → Search Service → Kafka → Analytics Service → Data Lake
                                                               │
                                                               ▼
                                                        Trie Builder
                                                               │
                                                               ▼
                                                            Redis
```

**Why Kafka?**
- Decouples search service from analytics
- High throughput for billions of events/day
- Durability and replay capability
- Multiple consumers can process same events

### 2. Building the Trie Data Structure

**Core Insight**: Use **Trie (Prefix Tree)** for fast prefix matching.

**Trie Properties**:
- Each node represents a character
- Path from root = prefix
- Leaf nodes = complete queries
- Store metadata at nodes: frequency, last updated

**Example Trie**:
```
        root
       /  |  \
      f   b   c
     /    |    \
    a     o     a
   /      |      \
  c       o       r
 /        |        \
e         k         s
|         |
book(5k)  booking(3k)
          |
          bookmark(2k)
```

**Storage**: Serialize trie and store in distributed cache (Redis)

**Building Process**:
1. Analytics service aggregates search data and writes to Data Lake
2. Trie Builder service reads aggregated query frequencies from Data Lake
3. Trie Builder constructs/updates trie structure
4. Each node stores: character, children pointers, top suggestions with scores
5. Trie Builder serializes trie to Redis (partitioned by prefix)

**Note**: Trie Builder is independent of Analytics Service - it only reads from Data Lake. This decoupling allows:
- Analytics can be updated without affecting trie building
- Trie Builder can run on its own schedule (hourly/daily)
- Multiple consumers can process Data Lake data independently

**Optimization**: Pre-compute top N suggestions at each node to avoid runtime sorting
- Node "fa" stores: ["facebook" (95k), "face masks" (45k), "fashion" (32k), ...]
- Node "fac" stores: ["facebook" (95k), "face masks" (45k), "factory" (12k), ...]

### 3. Serving Suggestions (Read Path)

**Components**:
- Load Balancer
- Autocomplete Service (stateless)
- Redis Cluster (distributed trie storage)
- User Profile Service (personalization)

**Diagram**:
```
┌──────────┐
│  Client  │ Types "fac"
└────┬─────┘
     │
     ▼
┌─────────────────┐ (L2 Cache - CDN)
│   CDN / Edge    │ Cache Hit? → Return suggestions
└────┬────────────┘
     │ Cache Miss
     ▼
┌─────────────────┐
│ Load Balancer   │
└────┬────────────┘
     │
     ▼
┌──────────────────────────────┐
│  Autocomplete Service        │
│  1. Extract userId from JWT  │
│  2. Query Redis for prefix   │──────┐
│  3. Fetch personalization    │      │
│  4. Merge & rank results     │      │
└──────────────────────────────┘      │
     │                │                │
     ▼                ▼                ▼
┌─────────────┐  ┌──────────────┐  ┌─────────────┐
│Redis Cluster│  │ DynamoDB+DAX │  │   Client    │
│   (Trie)    │  │ (User Hist)  │  │  (L1 Cache) │
└─────────────┘  └──────────────┘  └─────────────┘

Response: ["facebook", "face masks", "factory", ...]
```

**Flow**:
1. User types "fac" in search box
2. Client sends `GET /suggestions?prefix=fac&limit=5` (with auth token in header)
3. Autocomplete Service extracts userId from session/JWT token
4. Autocomplete Service queries Redis for "fac" node
5. Retrieve pre-computed top suggestions from node metadata
6. Fetch user's recent searches from User Profile Service (using userId)
7. Merge and rank: 70% popular + 30% personalized
8. Return top 5 suggestions in < 50ms

**Caching Strategy**:
- **L1 Cache** (Client-side): Cache last 100 prefixes locally for 5 minutes
- **L2 Cache** (CDN): Cache popular prefixes at edge locations
- **L3 Cache** (Redis): Distributed trie storage
- Cache hit ratio: 95%+ for popular prefixes

## Deep Dives

### 1. Trie Partitioning and Storage

**Problem**: Single trie for billions of queries won't fit in memory.

**Solution: Partition by Prefix**

**Diagram**:
```
Full Trie (30 GB) → Partitioned across Redis Cluster

┌─────────────────────────────────────────────────────┐
│              Consistent Hash Ring                   │
│                                                      │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│   │ Shard 0  │    │ Shard 1  │    │ Shard 2  │    │
│   │          │    │          │    │          │    │
│   │ a*, b*   │    │ c*, d*   │    │ e*, f*   │    │
│   │ (3 GB)   │    │ (3 GB)   │    │ (3 GB)   │    │
│   └─────┬────┘    └─────┬────┘    └─────┬────┘    │
│         │               │               │          │
│         ▼               ▼               ▼          │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│   │ Replica  │    │ Replica  │    │ Replica  │    │
│   │  Slave   │    │  Slave   │    │  Slave   │    │
│   └──────────┘    └──────────┘    └──────────┘    │
└─────────────────────────────────────────────────────┘

Query "facebook" → hash("fa") → Shard 2
Query "amazon" → hash("am") → Shard 0
```

**Partitioning Strategy**:
- Split trie into smaller tries based on first 1-2 characters
- "a*" → Trie Partition 0
- "b*" → Trie Partition 1
- "fa*" → Trie Partition 12
- etc.

**Consistent Hashing**:
- Hash prefix → Redis shard
- Each shard holds multiple trie partitions
- Adding shards only redistributes affected partitions

**Storage Estimation**:
- 1 billion unique queries
- Average query length: 50 bytes
- Trie compression: ~30 bytes/query (shared prefixes)
- Total: 30 GB for full trie
- Partitioned across 10 Redis shards: 3 GB/shard
- Easily fits in memory with replication

**Data Structure in Redis**:
```
Key: "trie:fac"
Value: {
  "children": ["e", "t", "u"],
  "suggestions": [
    {"text": "facebook", "score": 95000},
    {"text": "face masks", "score": 45000},
    {"text": "factory", "score": 12000}
  ]
}
```

**Replication**:
- Master-slave replication for high availability
- Read from slaves to distribute load
- 3 replicas per shard (1 master + 2 slaves)

### 2. Handling Updates (Write Path)

**Challenge**: Billions of searches/day → continuous trie updates. Can't rebuild entire trie on every search.

**Solution: Batch Updates + Incremental Refresh**

**Diagram**:
```
                    ┌─────────────────┐
Search Events ─────▶│     Kafka       │
                    └────┬───────┬────┘
                         │       │
                         │       └─────────────────────┐
                         │                             │
                         ▼                             ▼
              ┌──────────────────┐         ┌─────────────────┐
              │ Stream Processing│         │   Data Lake     │
              │  (Flink/Spark)   │         │   (S3/HDFS)     │
              │                  │         └────────┬────────┘
              │ • 1-min windows  │                  │
              │ • Detect spikes  │                  │
              │ • Trending boost │                  ▼
              └────────┬─────────┘         ┌─────────────────┐
                       │                   │ Batch Pipeline  │
                       │                   │  (MapReduce)    │
                       │                   │                 │
                       │                   │ • Daily aggr.   │
                       │                   │ • Build trie    │
                       │                   └────────┬────────┘
                       │                            │
                       └──────┬─────────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │   Redis Cluster     │
                   │  (Updated Trie)     │
                   │                     │
                   │ score = 0.8×batch   │
                   │       + 0.2×realtime│
                   └─────────────────────┘
```

**Architecture**:
1. **Real-time Pipeline** (seconds to minutes):
   - Stream processing (Spark Streaming / Flink)
   - Aggregates searches in 1-minute windows
   - Updates trending queries in Redis
   - Handles breaking news / viral events

2. **Batch Pipeline** (hours):
   - MapReduce job on data lake
   - Computes daily/weekly query frequencies
   - Rebuilds trie partitions
   - Updates Redis in background

**Update Flow**:
```
Search Events → Kafka → [Real-time] → Redis (trending)
                       ↓
                  Data Lake → [Batch] → Trie Builder → Redis (stable)
```

**Hybrid Scoring**:
- 80% weight: Historical popularity (batch)
- 20% weight: Recent trends (real-time)
- Formula: `score = 0.8 × historical_freq + 0.2 × recent_freq`

**Handling Spikes**:
- Celebrity death, breaking news → sudden query surge
- Real-time pipeline detects spike (e.g., 1000x normal)
- Boost score temporarily
- Gradually decay over hours/days

**Example**:
```
Normal day: "iPhone" → 50k searches/day
Breaking news: "iPhone 16 released" → 5M searches in 1 hour
Real-time boost: Bump "iPhone 16" to top suggestions immediately
Decay: Reduce boost over 24 hours back to organic ranking
```

### 3. Personalization

**Goal**: Mix global popular suggestions with user-specific history.

**Diagram**:
```
User types "fac"
     │
     ▼
┌──────────────────────┐
│ Autocomplete Service │
└─────┬───────────┬────┘
      │           │
      │           │ (Parallel requests)
      │           │
      ▼           ▼
┌─────────┐  ┌────────────────┐
│  Redis  │  │  DynamoDB+DAX  │
│ (Trie)  │  │ (User History) │
│         │  │                │
│ Global  │  │ userId: 123    │
│ Popular │  │ ┌────────────┐ │
│         │  │ │"facebook"  │ │
│"facebook│  │ │"face rec"  │ │
│ (95k)"  │  │ │"factory"   │ │
│"face    │  │ └────────────┘ │
│ masks   │  │  (prefix=fac)  │
│ (45k)"  │  └────────────────┘
└─────────┘
      │           │
      └─────┬─────┘
            ▼
     ┌─────────────┐
     │   Merge &   │
     │    Rank     │
     │             │
     │ 70% Global  │
     │ 30% Personal│
     └──────┬──────┘
            │
            ▼
     ["facebook" (trending),
      "facebook login" (popular),
      "face recognition" (personal),
      "face masks" (popular),
      "factory" (personal)]
```

**Data Storage**:
- **User Search History**: DynamoDB
  - Partition key: userId
  - Sort key: timestamp
  - TTL: 90 days (privacy)
  - Attributes: query, clicked, timestamp

**Personalization Strategy**:
1. Fetch user's recent searches (last 30 days)
2. Filter by prefix match
3. Score based on recency and frequency
4. Merge with global suggestions:
   - 3-4 slots: Global popular (from trie)
   - 1-2 slots: Personalized (user history)

**Privacy Considerations**:
- Incognito mode: Skip personalization
- User can clear history: Delete DynamoDB entries
- TTL ensures automatic expiry

**Fallback**:
- New user (no history): 100% global suggestions
- User history service down: Serve global only (graceful degradation)

### 4. Achieving < 100ms Latency

**Latency Breakdown** (Target: 50ms total):
- Network: 10-20ms (CDN/edge servers)
- Load balancer: 1ms
- Application logic: 5ms
- Redis lookup: 1-2ms (in-memory)
- Personalization query: 5ms (DynamoDB with DAX cache)
- Merge & rank: 3ms
- Response serialization: 2ms

**Optimizations**:

1. **Geographic Distribution**:
   - Deploy autocomplete service in multiple regions
   - Route to nearest region (latency-based routing)
   - Replicate Redis across regions

2. **Pre-computation**:
   - Store top N suggestions at every trie node
   - No runtime sorting needed
   - Trade space for speed

3. **Async Personalization**:
   - Fetch global suggestions first (cache hit)
   - Fetch personalization in parallel
   - Merge when both arrive
   - If personalization slow (>20ms), timeout and return global only

4. **Client-Side Optimization**:
   - Debounce keystrokes (wait 100-200ms before API call)
   - Cancel in-flight requests when user types more
   - Local cache for recent prefixes

5. **Request Collapsing**:
   - If 1000 users type "fac" simultaneously
   - Collapse to single Redis lookup
   - Broadcast result to all waiting requests
   - Reduces load by 1000x for popular prefixes

### 5. Scaling to Billions of Searches

**Component Scaling**:

**Autocomplete Service**:
- Stateless, horizontally scalable
- Auto-scaling based on request rate
- 100k requests/second → ~50-100 servers

**Redis Cluster**:
- Consistent hashing for trie partitions
- 10-20 shards for 30-60 GB trie
- 3x replication (1 master + 2 replicas)
- Read from replicas to distribute load

**Data Lake (S3/HDFS)**:
- Stores raw search logs (petabytes)
- Partitioned by date
- Compressed (Parquet/ORC)
- Accessed by batch jobs only

**Kafka**:
- Topic partitioning by userId hash
- 100+ brokers for billions of events/day
- Retention: 7 days
- Multiple consumer groups (analytics, fraud detection, etc.)

**Analytics Service**:
- Spark/Flink clusters for stream processing
- MapReduce for batch aggregation
- Elastic compute (scale based on data volume)

**Traffic Patterns**:
- Peak: 10M concurrent users
- Average: 100k requests/second
- Peak: 500k requests/second
- Auto-scaling handles spikes

**Cost Optimization**:
- CDN caching reduces backend calls by 80%
- Client-side caching reduces API calls by 50%
- Batch processing on cheap spot instances
- Cold data (>30 days) archived to Glacier

### 6. Handling Typos and Misspellings

**Problem**: User types "facbook" instead of "facebook".

**Solutions**:

1. **Fuzzy Matching**:
   - Edit distance (Levenshtein) calculation
   - Allow 1-2 character difference
   - "facbook" → check "facebook" (1 edit away)
   - Too slow for real-time at scale

2. **Pre-computed Corrections** (Recommended):
   - Build dictionary of common misspellings
   - Store in Redis: `misspelling → correct_queries[]`
   - Example: `"facbook" → ["facebook", "face book"]`
   - Lookup during suggestion generation
   - Merge corrected suggestions with regular ones

3. **Phonetic Matching**:
   - Soundex/Metaphone algorithms
   - "facebook" and "fasebook" sound similar
   - Convert to phonetic code, lookup variants

**Implementation**:
```
User types: "facbook"
1. Lookup trie for "facbook" → few/no results
2. Lookup correction dictionary → "facebook"
3. Fetch suggestions for "facebook"
4. Return with "Did you mean: facebook" flag
```

**Building Correction Dictionary**:
- Analyze search logs for query corrections
- User types "facbook", then "facebook" → learn mapping
- Aggregate millions of correction pairs
- Store top 10k common misspellings

## Key Learnings & Insights

### Design Patterns

**Pattern: Trie for Prefix Matching**:
- O(k) lookup time where k = prefix length
- Shared prefixes save memory
- Pre-computation at nodes for speed
- Applies to: Autocomplete, IP routing, dictionary lookups

**Pattern: Lambda Architecture**:
- Batch layer: Slow, accurate, complete data
- Speed layer: Fast, approximate, recent data
- Serving layer: Merges both for queries
- Applies to: Any system needing real-time + historical data

**Pattern: Write-Heavy to Read-Heavy Conversion**:
- Billions of writes (searches) → aggregated → millions of unique queries
- Write to message queue → batch process → serve from read-optimized store
- Trading freshness for performance

### Scaling Insights

**Read vs Write Optimization**:
- Write path: Buffer in Kafka, batch process, eventual updates
- Read path: Multi-layer caching, pre-computation, geographic distribution
- 1000:1 read-to-write ratio for suggestion fetches vs updates

**Caching is Critical**:
- Popular prefixes ("a", "f", "g") hit constantly
- 95%+ cache hit rate at CDN/client
- Remaining 5% served from Redis (still fast)
- No database queries on read path

**Geographic Distribution**:
- "facebook" popular in US, "flipkart" in India
- Region-specific tries for better relevance
- Edge servers serve local popular queries
- Reduces latency by 50-100ms

### Data Structure Insights

**Why Trie over Alternatives**:
- ❌ SQL LIKE queries: Too slow, can't scale
- ❌ Elasticsearch prefix queries: Overkill, higher latency
- ❌ Hash map: No prefix matching support
- ✅ Trie: Purpose-built for prefix matching, fast, memory-efficient

**Trie Optimizations**:
- Compress single-child chains: "a" → "u" → "t" → "o" becomes "auto"
- Store only top N at each node (not all children)
- Lazy loading: Load trie partitions on-demand
- Memory-mapped files for faster cold starts

### Interview Strategy

**Common Pitfalls**:
- Building trie in real-time on every search (too slow)
- Not considering cache layers
- Ignoring geographic latency
- No strategy for updates (stale data)
- Missing personalization vs popularity balance

**What Interviewers Look For**:
- Understanding of trie data structure and why it fits
- Multi-layer caching strategy
- Separation of read and write paths
- Real-time + batch processing (Lambda architecture)
- Latency breakdown and optimization techniques
- Scalability discussion (partitioning, replication)

**Deep Dive Selection**:
- Start with data structure choice (trie)
- Move to read path optimization (caching)
- Then write path (updates, trending)
- End with advanced topics (personalization, typos, scale)

### Architecture Principles

**Separation of Concerns**:
- Collection: Search service → Kafka
- Processing: Analytics → Trie builder
- Storage: Redis (serving) vs S3 (archival)
- Serving: Autocomplete service (stateless)

**Asynchronous Processing**:
- Don't block users on logging
- Fire-and-forget to Kafka
- Background jobs update tries
- Users see updated suggestions eventually

**Graceful Degradation**:
- Personalization service down → serve global suggestions
- Redis shard down → fallback to replicas or CDN cache
- Trending pipeline broken → serve stable batch suggestions
- Always return something useful

## Additional Considerations

**Optimizations**:
- **Smart Prefetching**: Predict next character, pre-fetch suggestions
- **Compression**: Gzip/Brotli response bodies (5-10x smaller)
- **Connection Pooling**: Reuse HTTP connections for low overhead
- **Bloom Filters**: Check if prefix exists before trie lookup (negative cache)

**Advanced Features**:
- **Rich Suggestions**: Include images, ratings, metadata
- **Query Segmentation**: "new york weather" → "new york" + "weather"
- **Spell Correction**: Integrate with spell checker service
- **Voice Input**: Phonetic matching for voice searches
- **Multi-language**: Separate tries per language
- **Banned Queries**: Filter offensive/illegal suggestions

**Monitoring & Metrics**:
- Latency percentiles (p50, p95, p99)
- Cache hit rates (client, CDN, Redis)
- Suggestion click-through rates
- Trie update lag (freshness)
- Error rates and fallback usage

**Privacy & Compliance**:
- Don't log queries in incognito mode
- GDPR: User data deletion support
- TTL on personal search history
- Anonymize logs after aggregation

**A/B Testing**:
- Test suggestion ranking algorithms
- Personalization weight tuning
- UI changes (number of suggestions, display format)
- Measure click-through rate improvements

## Technology Choices

- **Trie Storage**: Redis Cluster (in-memory, partitioned, replicated)
- **Message Queue**: Apache Kafka (high throughput, durability)
- **Batch Processing**: Apache Spark / Hadoop MapReduce
- **Stream Processing**: Apache Flink / Spark Streaming
- **Data Lake**: AWS S3 / Hadoop HDFS (Parquet format)
- **User History**: DynamoDB with DAX (fast key-value lookups)
- **CDN**: CloudFront / Akamai (edge caching)
- **Load Balancer**: AWS ALB / NGINX (geographic routing)
- **Monitoring**: Prometheus + Grafana, DataDog

## System Diagram

```
┌─────────────┐
│   Clients   │ (Web/Mobile)
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│   CDN / Edge    │ (Cache popular prefixes)
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ Load Balancer   │ (Geographic routing)
└──────┬──────────┘
       │
       ▼
┌──────────────────────────────┐
│  Autocomplete Service        │ (Stateless, horizontally scaled)
└──────┬───────────────┬───────┘
       │               │
       ▼               ▼
┌─────────────┐  ┌──────────────┐
│Redis Cluster│  │ DynamoDB+DAX │ (User history)
│   (Trie)    │  │              │
└─────────────┘  └──────────────┘
       ▲
       │
┌──────────────────┐
│  Trie Builder    │ (Background jobs)
└──────┬───────────┘
       │
       ▼
┌────────────────────────┐
│  Analytics Pipeline    │
│  ┌─────────────────┐   │
│  │ Stream (Flink)  │   │ (Real-time trending)
│  └─────────────────┘   │
│  ┌─────────────────┐   │
│  │ Batch (Spark)   │   │ (Daily aggregation)
│  └─────────────────┘   │
└────────┬───────────────┘
         │
         ▼
┌─────────────────┐
│   Kafka         │ (Search event stream)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Data Lake      │ (S3/HDFS - historical logs)
└─────────────────┘
         ▲
         │
┌────────────────┐
│ Search Service │ (Logs queries)
└────────┬───────┘
         │
         ▼ (from users)
```

## Capacity Estimation

**Assumptions**:
- 1 billion searches/day
- 100 million unique queries
- Average query length: 50 bytes
- Peak: 50k requests/second
- Storage retention: 30 days active, 1 year archive

**Storage**:
- Trie: 100M queries × 30 bytes = 3 GB (with compression)
- User history: 1B users × 100 queries × 50 bytes = 5 TB
- Search logs: 1B searches/day × 100 bytes × 30 days = 3 TB (active)
- Archive: 3 TB × 12 months = 36 TB/year

**Bandwidth**:
- Inbound (search logging): 1B × 100 bytes / 86400s = 1.2 MB/s
- Outbound (suggestions): 10B requests/day × 500 bytes = 58 MB/s
- With CDN cache (80% hit): 12 MB/s to origin

**Compute**:
- Autocomplete service: 50k req/s ÷ 1k req/s/server = 50 servers
- Redis: 10 shards × 3 replicas = 30 instances (16 GB each)
- Kafka: 20 brokers (high throughput)
- Analytics: 50 Spark executors (batch), 20 Flink workers (stream)

**Cost** (monthly estimate):
- Compute: $10k (autocomplete + analytics)
- Storage: $2k (Redis + DynamoDB + S3)
- Network: $5k (CDN + data transfer)
- Total: ~$20k/month for 1B searches/day
