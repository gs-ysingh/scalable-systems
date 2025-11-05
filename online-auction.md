# Online Auction System Design

## Requirements

### Functional
- ✅ Post items for auction (starting price, end date)
- ✅ Bid on items (must be higher than current max bid)
- ✅ View auction with current highest bid
- ⬜ Search, filter, sort items

### Non-Functional
- ✅ Strong consistency for bids
- ✅ Fault tolerance & durability (no dropped bids)
- ✅ Real-time bid updates
- ✅ Scale: 10M concurrent auctions, ~1B bids/day (10K bids/sec)

### Scale Assumptions
- 10M concurrent auctions
- ~100 bids per auction
- 1B bids/day = 10K bids/sec peak

## Core Entities

**Auction**: startingPrice, endDate, currentMaxBid, itemId  
**Item**: name, description, imageUrl  
**Bid**: amount, userId, auctionId, status (accepted/rejected), timestamp  
**User**: userId, profile info

**Key Decision**: Separate Item from Auction for reusability and independent updates.

## API Design

```
POST /auctions -> Auction & Item
GET /auctions/:auctionId -> Auction & Item  
POST /auctions/:auctionId/bids -> Bid
```

## Architecture Overview

### Components

**Client** → **API Gateway** → **Services** → **Database/Cache**

**Auction Service**: CRUD for auctions and items  
**Bidding Service**: Validate bids, update max bid, store bid history (100x more traffic than auctions)

**Why separate services?**
- Independent scaling (bidding is 100x higher volume)
- Isolated business logic (complex validation, race conditions)
- Different optimization needs (write-heavy vs read-heavy)

---

## Deep Dive: Strong Consistency for Bids

### The Problem: Race Conditions

Current max bid = $10
- User A bids $100 (valid)
- User B reads stale $10, bids $20 (should be rejected but gets accepted)
- Result: Both think they're winning ❌

### Solution Evolution

**❌ Naive: Lock all bids rows** → Too slow, locks too many rows

**❌ Cache max bid** → Fast, but cache-DB consistency issues

**✅ Store max bid in Auction table**

```sql
-- Pessimistic locking (simple)
BEGIN TRANSACTION;
  SELECT max_bid FROM auctions WHERE id = :id FOR UPDATE; -- Lock 1 row
  INSERT INTO bids (...);
  UPDATE auctions SET max_bid = :new_bid WHERE id = :id;
COMMIT;
```

**✅ Optimistic Concurrency Control (OCC)** - Better for low contention

```sql
-- No locks, just version checking
UPDATE auctions
SET max_bid = :new_bid
WHERE id = :auction_id AND max_bid = :original_max_bid;
-- If 0 rows updated, max_bid changed → retry
```

**Key Insight**: OCC ideal since bid conflicts are rare. Retry on conflict.

---

## Deep Dive: Fault Tolerance & Durability

### Pattern: Message Queue for Durability

**Problem**: Cannot drop bids. System must survive crashes and load spikes.

**Solution**: Kafka message queue

```
Client → API Gateway → Kafka Producer → Kafka → Bid Service Consumer → DB
```

**Benefits**:

1. **Durability**: Bid written to Kafka immediately (disk + replication)
2. **Buffer against spikes**: Queue absorbs 1000s of bids/sec, consumers process at their pace
3. **Guaranteed ordering**: First bid wins (partition by auctionId)

**Why Kafka?**
- Millions of msgs/sec throughput
- Disk persistence + replication
- Partitioning for parallel processing

**Tradeoff**: +2-10ms latency for durability guarantee

---

## Deep Dive: Real-Time Bid Updates

### Pattern: Real-time Updates via SSE

**❌ Polling**: Inefficient (most requests return no change), slow (seconds delay)

**✅ Server-Sent Events (SSE)**: Unidirectional server→client push

**Client**:

```javascript
const eventSource = new EventSource(`/api/auctions/${auctionId}/bid-stream`);
eventSource.onmessage = (event) => {
  const { maxBid } = JSON.parse(event.data);
  updateUI(maxBid);
};
```

**Server**:

```typescript
class AuctionEventManager {
  private connections: Map<string, Set<Connection>> = new Map();
  
  broadcastNewBid(auctionId: string, maxBid: number) {
    const connections = this.connections.get(auctionId);
    connections?.forEach(conn => 
      conn.write(`data: ${JSON.stringify({ maxBid })}\n\n`)
    );
  }
}
```

**Why SSE over WebSockets?** Unidirectional, lighter weight, simpler for this use case.

### Scaling Challenge: Multiple Servers

**Problem**: User A on Server 1, User B on Server 2 watching same auction. Bid goes to Server 1 → User B doesn't get update.

**Solution**: Pub/Sub (Redis/Kafka)

```
Bid Service → Publish bid → Redis Pub/Sub → All Bid Service instances subscribe → Push to their SSE connections
```

---

## Scaling to 10M Auctions

### Capacity Planning

**Throughput**: 10M auctions × 100 bids = 1B bids/day = **10K bids/sec**

**Storage**: 10M × 52 weeks = 520M auctions/year  
520M × (1KB auction + 50KB bids) = **25TB/year**

### Scaling Each Component

**✅ Message Queue (Kafka)**: 10K/sec easily handled, no partitioning needed

**✅ Bid Service / Auction Service**: Horizontal scaling + auto-scaling (stateless)

**✅ Database**:
- Storage: 25TB/year manageable with modern SSDs
- **Write throughput bottleneck**: 10K writes/sec at limit for single Postgres
- **Solution**: Shard by auctionId (no scatter-gather since all ops for auction on same shard)

**✅ SSE Connections**: 100M users watching auctions across servers
- **Solution**: Redis Pub/Sub to broadcast bids across all Bid Service instances

---

## Additional Considerations

**Dynamic auction end times**: Update auction.endTime on each bid. Cron job finds expired auctions. For precision: SQS with visibility timeout + worker checks if bid still winning.

**Purchasing**: Email winner → payment within N hours → else go to next highest bidder.

**Bid history**: Same as real-time updates (SSE + Pub/Sub).

**Images**: Store in blob storage (S3), reference by URL in items table.

---

## Key Learnings Summary

1. **Dealing with Contention Pattern**: Use OCC or pessimistic locking with minimal row locks
2. **Message Queue for Durability**: Kafka provides fault tolerance + buffering + ordering
3. **Real-time Updates Pattern**: SSE for server→client push, Pub/Sub for multi-server coordination
4. **Consistency First**: Never destroy historical data (bids table, not just maxBid field)
5. **Separate Services by Load Profile**: Bidding (write-heavy, high volume) vs Auctions (read-heavy)
6. **Sharding Strategy**: By auctionId to avoid scatter-gather while distributing load
