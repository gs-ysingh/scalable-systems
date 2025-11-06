# Rate Limiter System Design

## Problem Statement

A rate limiter controls the number of requests a client can make within a specific timeframe. It prevents abuse, protects servers from traffic bursts, and ensures fair usage. When limits are exceeded, it returns HTTP 429 "Too Many Requests".

## Requirements

### Functional

- Identify clients by user ID, IP address, or API key
- Limit requests based on configurable rules (e.g., 100 requests/minute per user)
- Return HTTP 429 with helpful headers when limits exceeded

### Non-Functional

- **Scale**: 1M requests/second, 100M daily active users
- **Latency**: < 10ms overhead per request
- **Availability**: High availability with eventual consistency

### Out of Scope

- Analytics/querying on rate limit data
- Long-term persistence
- Strong consistency guarantees

## Core Components

### Entities

- **Rules**: Rate limiting policies (requests per time window, client type, endpoints)
- **Clients**: Entities being limited (users, IPs, API keys)
- **Requests**: Incoming API requests to evaluate

### API Interface

```typescript
isRequestAllowed(clientId, ruleId) -> { passes: boolean, remaining: number, resetTime: timestamp }
```

## Design

### 1. Placement: API Gateway

Place rate limiter at the API Gateway for:

- Centralized control
- No extra network hops
- Access to request headers (Authorization, X-API-Key, X-Forwarded-For)

### 2. Client Identification

- **User ID**: For authenticated APIs (from JWT in Authorization header)
- **IP Address**: For public APIs (from X-Forwarded-For header) - note NAT issues
- **API Key**: For developer APIs (from X-API-Key header)

**Layered Rules** - Apply multiple rules simultaneously:

- Per-user: 1000 requests/hour
- Per-IP: 100 requests/minute  
- Per-endpoint: Search API 10/min, profile updates 100/min

Enforce the most restrictive rule.

### 3. Rate Limiting Algorithm

We'll use **Token Bucket** - best balance of simplicity, memory efficiency, and handling bursty traffic.

#### Algorithm Comparison

1. **Fixed Window Counter**: Simple hash table `{clientId:window => count}`. Issue: boundary effects
2. **Sliding Window Log**: Stores timestamps. Perfect accuracy but high memory
3. **Sliding Window Counter**: Weighs current + previous windows. Lower memory, approximation
4. **Token Bucket** ✓: Bucket refills at steady rate, consumes token per request. Handles bursts + sustained load

### 4. Storage: Redis

**Problem**: Multiple API gateways need shared state to track user buckets.

**Solution**: Redis as centralized data store.

**Implementation Flow**:

1. Request arrives for user `alice`
2. Gateway fetches: `HMGET alice:bucket tokens last_refill`
3. Calculate tokens to add based on elapsed time
4. Update atomically via Lua script (prevents race conditions):

   ```redis
   MULTI
   HSET alice:bucket tokens <new_count>
   HSET alice:bucket last_refill <timestamp>
   EXPIRE alice:bucket 3600
   EXEC
   ```

5. Allow if tokens ≥ 1, else reject

**Why Redis**:

- Sub-millisecond latency
- Auto-cleanup via EXPIRE
- Atomic operations (Lua scripts for read-modify-write)
- High availability via replication

#### Token Bucket Flow Explained

**Setup**: Each user (e.g., Alice) has a bucket storing:

- `tokens` → remaining requests allowed
- `last_refill` → timestamp of last update

**Step-by-step**:

1. **Request arrives**: Alice makes an API request

2. **Fetch state**: Gateway queries Redis

   ```redis
   HMGET alice:bucket tokens last_refill
   # Returns: { tokens: 4, last_refill: 100 }
   ```

3. **Refill calculation**: Add tokens based on elapsed time
   - If 30 seconds passed and refill rate is 1 token/sec
   - New tokens = `min(4 + 30, max_capacity)` → 34 tokens

4. **Update state**: Save new values atomically

   ```redis
   MULTI
   HSET alice:bucket tokens 34
   HSET alice:bucket last_refill 130
   EXPIRE alice:bucket 3600
   EXEC
   ```

5. **Decision**: Allow request if tokens ≥ 1, decrement counter. Else reject with 429.

### 5. HTTP 429 Response

**Fail fast** - reject immediately (don't queue)

**Response headers**:

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995200
Retry-After: 60
```

## Scaling Challenges

### 1. Horizontal Scaling (1M req/sec)

**Problem**: Single Redis handles ~100k ops/sec. We need 10x capacity.

**Solution**: Redis Cluster with sharding

- Hash client ID (user/IP/API key) to determine shard
- Consistent hashing ensures same client → same shard
- 10 shards × 100k ops/sec = 1M req/sec capacity

Use **Redis Cluster** for automatic sharding across 16,384 hash slots.

### 2. High Availability

**Failure modes**:

- **Fail-open**: Allow all requests (risk overwhelming backend)
- **Fail-closed**: Reject all requests (safer for critical systems) ✓

**Solution**: Master-replica replication

- Each shard has read replicas
- Auto-failover promotes replica when master fails
- Redis Cluster handles this automatically

**Monitoring**: Track Redis health, rate limit success rates, latency

### 3. Latency Optimization

**Techniques**:

- **Connection pooling**: Reuse TCP connections (avoid 20-50ms handshake)
- **Geographic distribution**: Deploy Redis clusters per region
- Accept eventual consistency across regions

### 4. Hot Keys (DDoS/High-Volume Clients)

**For legitimate clients**:

- Client-side rate limiting (respect headers)
- Request batching
- Premium tiers with higher limits

**For abusive traffic**:

- Auto-block after N violations
- DDoS protection (Cloudflare, AWS Shield)
- Set higher IP limits (account for NAT/public WiFi)

### 5. Dynamic Configuration

**Need**: Adjust limits without redeploying code

**Approaches**:

- **Config service**: Store rules in database, cache in-memory, poll for updates
- **Admin API**: Runtime updates to rate limit rules
- **Feature flags**: Enable/disable or modify limits per user tier

## Summary

**Architecture**: API Gateway → Redis Cluster → Backend Services

**Key Decisions**:

- **Algorithm**: Token Bucket (handles bursts, memory efficient)
- **Storage**: Redis with Lua scripts (atomic operations)
- **Scaling**: Consistent hashing across 10+ shards
- **Availability**: Master-replica with fail-closed mode
- **Identification**: Multi-layered (User ID + IP + API Key)

**Response**: HTTP 429 with X-RateLimit-* headers
