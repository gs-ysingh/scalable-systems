# URL Shortener (Bit.ly) System Design

## Requirements

### Functional
- ✅ Shorten long URLs → short URLs
- ✅ Custom aliases (optional)
- ✅ Expiration dates (optional)
- ✅ Redirect short → long URLs
- ❌ User auth, analytics (out of scope)

### Non-Functional
- **Uniqueness**: No collision between short codes
- **Latency**: <100ms redirects
- **Availability**: 99.99% uptime (favor availability > consistency)
- **Scale**: 1B URLs, 100M DAU
- **Read-heavy**: 1000:1 read-to-write ratio

## Core Entities
- **Original URL**: Long URL to be shortened
- **Short URL**: Generated short code
- **User**: Creator of the shortened URL

## API Design

```
POST /urls
{
  "long_url": "https://example.com/very/long/url",
  "custom_alias": "optional",
  "expiration_date": "optional"
}
→ { "short_url": "http://short.ly/abc123" }

GET /{short_code}
→ HTTP 302 Redirect to original URL
```

**Key Decision**: Use 302 (temporary) vs 301 (permanent) redirect
- **302 is preferred** - allows tracking, updates, and prevents browser caching

## High-Level Design

### Creating Short URLs
1. **Client** → POST request with long URL
2. **Primary Server**:
   - Validates URL (using libraries like `is-url`)
   - Checks for duplicates in DB
   - Generates unique short code
   - Validates custom alias if provided
3. **Database**: Stores mapping (short_code, long_url, expiration, custom_alias)
4. Returns short URL to client

### Redirecting Users
1. User visits `short.ly/abc123`
2. Server looks up `abc123` in database
3. Checks if not expired
4. Returns HTTP 302 redirect to original URL

## Key Learnings

### 1. Ensuring Uniqueness of Short URLs
**Constraints**:
- Codes must be unique
- As short as possible
- Efficiently generated

**Solution Options**:
- Base62 encoding of auto-incrementing counter
- Hash functions (with collision handling)
- Random generation (with retry logic)
- Pre-generated key pools

### 2. Fast Redirects (Scaling Reads)
**Problem**: Full table scans are too slow at scale

**Solutions**:
- **Database indexing** on short_code column
- **Caching layer** (Redis/Memcached) for hot URLs
  - Cache popular URLs to avoid DB hits
  - LRU eviction policy
  - Read-through cache pattern

**Impact**: Handles extreme read-heavy workload (1000:1 ratio)

### 3. Scaling to 1B URLs & 100M DAU

#### Database Sizing
- Per row: ~500 bytes (short_code + long_url + metadata)
- 1B URLs = 500GB (fits on modern SSDs)
- Write rate: ~100k/day = ~1/second
- **Database choice**: Postgres, MySQL, or DynamoDB all work
  - Pick based on familiarity
  - Postgres is a safe default

#### High Availability
- **Database replication**: Multiple read replicas
- **Periodic backups**: Snapshots for disaster recovery

#### Microservice Architecture
Separate **Read Service** and **Write Service**:
- Scale independently based on demand
- Read Service handles redirects (high volume)
- Write Service handles URL creation (low volume)

#### Counter Management Problem
When horizontally scaling Write Service, need **global counter uniqueness**:

**Solution**: Centralized Redis counter
- Single-threaded, atomic increments
- All Write Service instances get next value from Redis
- Supports distributed uniqueness

**Optimization**: Counter batching
- Request batches of 1000 values at once
- Reduces network calls to Redis
- Each instance uses batch locally before requesting more
- Redis replication for high availability

## Architecture Summary

```
Client → Load Balancer
         ├→ Read Service (scaled) → Cache → Database
         └→ Write Service (scaled) → Redis Counter → Database
```

**Key Patterns**:
- Read-write separation
- Aggressive caching for reads
- Centralized counter for write uniqueness
- Horizontal scaling with load balancing
- Database replication for availability