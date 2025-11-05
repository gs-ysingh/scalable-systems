# Uber System Design 🚗

## What is Uber?
Ride-sharing platform connecting passengers with drivers for on-demand transportation.

## Requirements

### Functional Requirements

**Core Features (Above the Line):**
- ✓ Get fare estimate (start → destination)
- ✓ Request ride based on fare
- ✓ Match with nearby available driver
- ✓ Driver accept/decline & navigate

**Below the Line:**
- Ratings, scheduling, ride categories

### Non-Functional Requirements

- Low latency matching (< 1 min)
- Strong consistency (no double-booking drivers)
- High throughput (100k requests from same location)

**Key Learning**: Prioritize top 3 functional requirements. Everything else is "below the line" to maintain focus.

---

## Core Entities

**Rider**
- name
- contact
- payment

**Driver**
- name
- vehicle
- status

**Fare**
- pickup
- destination
- price
- eta

**Ride**
- riderId
- driverId
- fareId
- status
- route

**Location**
- driverId
- lat/long
- timestamp

**Key Learning**: Start with high-level entities. Details come later. Build sequentially through functional requirements.

---

## API Design

**1. Get Fare Estimate**
- **Endpoint:** `POST /fare`
- **Body:** `{ pickupLocation, destination }`
- **Response:** `Fare { price, eta, fareId }`

**2. Request Ride**
- **Endpoint:** `POST /rides`
- **Body:** `{ fareId }`
- **Response:** `Ride { rideId, status, driver }`

**3. Update Driver Location**
- **Endpoint:** `POST /drivers/location`
- **Body:** `{ lat, long }`
- **Note:** driverId from session/JWT
- **Response:** `Success/Error`

**4. Driver Accept/Decline**
- **Endpoint:** `PATCH /rides/:rideId`
- **Body:** `{ accept/deny }`
- **Response:** `Ride { pickup, destination }`

**Key Learning**: Never trust client data. User info comes from session/JWT. Timestamps generated server-side. No fare estimates from client.

---

## High-Level Design

### 1. Fare Estimation

```
┌─────────┐      ┌────────────┐      ┌──────────────┐
│ Rider   │─────>│ API        │─────>│ Ride         │
│ Client  │      │ Gateway    │      │ Service      │
└─────────┘      └────────────┘      └──────────────┘
                                            │
                                            ↓
                                     ┌──────────────┐
                                     │ Maps API     │
                                     │ (distance/   │
                                     │  time calc)  │
                                     └──────────────┘
                                            │
                                            ↓
                                     ┌──────────────┐
                                     │ Database     │
                                     │ (save Fare)  │
                                     └──────────────┘
```

**Flow**: Rider → Gateway → Ride Service → Maps API → Calculate Fare → Save to DB → Return to Client

**Key Learning**: Abstract complex pricing logic. Use third-party APIs for mapping/routing.

---

### 2. Request Ride

**Flow:**
1. Rider Client sends fareId → API Gateway
2. API Gateway → Ride Service
3. Ride Service creates Ride record in Database
   - Status: "requested"
   - Links to fareId
4. Trigger matching workflow

---

### 3. Driver Matching

```
┌──────────┐                    ┌──────────────┐
│ Driver   │──(every 5s)──────>│ Location     │
│ Client   │  lat/long          │ Service      │
└──────────┘                    └──────────────┘
                                       │
                                       ↓
                                ┌──────────────┐
                                │ Database     │
                                │ Update       │
                                │ Location     │
                                └──────────────┘
                                       ↑
                                       │ (query nearby)
                                       │
┌──────────┐      ┌────────────┐      ┌──────────────┐
│ Ride     │─────>│ Matching   │─────>│ Find closest │
│ Request  │      │ Service    │      │ available    │
└──────────┘      └────────────┘      │ drivers      │
                         │             └──────────────┘
                         ↓
                  ┌──────────────┐
                  │ Notification │
                  │ Service      │
                  └──────────────┘
                         │
                         ↓ (APNs/FCM)
                  ┌──────────────┐
                  │ Driver       │
                  │ Client       │
                  └──────────────┘
```

**Flow**: Drivers update location → Matching service queries nearby → Notify top driver

**Key Learning**: Separate location updates from matching logic. Use push notifications (APNs/FCM).

---

### 4. Driver Accept/Decline

**Accept Flow:**
1. Driver accepts → API Gateway
2. API Gateway → Ride Service
3. Ride Service updates Ride in Database
   - Status: "accepted"
   - Add driverId
4. Return pickup coordinates to Driver
5. Driver navigates using GPS

**Decline Flow:**
- Move to next driver in ranked list
- Repeat matching process

---

## Deep Dives

### 1. Location Updates & Proximity Search

#### The Problem Explained

Imagine we have 10 million drivers, and each one sends their location every 5 seconds. That's **2 million location updates every second**! 

**Why traditional databases fail:**

1. **Write Overload**: 
   - Regular databases like PostgreSQL or DynamoDB aren't built for this volume
   - DynamoDB cost: ~$100,000 per day just for location updates!
   - Database would slow to a crawl or crash

2. **Finding Nearby Drivers is Slow**:
   - Without optimization, we'd check EVERY driver's location
   - Calculate distance from rider to each of 10 million drivers
   - Full table scan - extremely inefficient at scale

#### Solution: Geospatial Indexing

Think of geospatial indexing like organizing a library. Instead of searching every book, we organize by sections, then shelves, then specific locations.

**Option 1: QuadTree / Geohash**

Divide the world map into grid cells (like a checkerboard):

**How it works:**
- World Map divided into cells: A1, A2, B1, B2, etc.
- Each cell has a unique ID
- Driver location = Cell ID + exact coordinates

**Example Grid:**
- Cell A1: 🚗 🚗 (2 drivers)
- Cell A2: (empty)
- Cell B1: (empty)
- Cell B2: 🚗 (1 driver)

**When rider requests in Cell A1:**
1. Only check drivers in A1
2. Check neighboring cells (A2, B1)
3. Ignore distant cells (B2)

**Result:** Search hundreds of drivers, not millions!

**Geohash Example:**
- San Francisco downtown: "9q8yy"
- Nearby location: "9q8yz"
- Similar prefix = nearby!

**Benefits**:
- Only search relevant area (not entire world)
- Updates just change cell membership
- Can use regular database indexes on cell IDs

**Option 2: Redis Geo Commands**

Redis has built-in geographic features - like having a GPS built into your database:

**Store driver location:**
```
GEOADD drivers 122.4 37.8 driver123
(longitude, latitude, driver ID)
```

**Find nearby drivers:**
```
GEORADIUS drivers 122.4 37.8 5 km
Returns: [driver123, driver456, ...]
```

**Auto-cleanup old locations:**
```
SET driver:123:location ... EX 30
(expires after 30 seconds if not updated)
```

**Why Redis?**
- In-memory = super fast reads/writes
- Built-in geo commands (no custom code)
- TTL cleans up inactive drivers
- Handles 2M writes/sec easily

**Benefits**:
- Simple to use (one command finds nearby drivers)
- Very fast (everything in memory)
- Automatic cleanup of stale data

**Option 3: PostGIS (PostgreSQL Extension)**

Add geographic superpowers to PostgreSQL:

**Create spatial column:**
```sql
ALTER TABLE drivers
ADD COLUMN location GEOGRAPHY(POINT);
```

**Create spatial index (R-tree):**
```sql
CREATE INDEX ON drivers
USING GIST(location);
```

**Find nearby drivers:**
```sql
SELECT * FROM drivers
WHERE ST_DWithin(
  location,
  ST_MakePoint(122.4, 37.8),
  5000  -- 5km in meters
);
```

**R-tree index** = Smart spatial search (like a tree of geographic regions)

**Benefits**:
- Persistent storage (survives restarts)
- ACID transactions (data consistency)
- Rich query capabilities

#### Recommended Approach

**Hybrid Solution (Best of Both):**

**1. Redis Geo for real-time queries**
- Active driver locations
- Fast proximity searches
- High write throughput

**2. PostgreSQL/PostGIS for history**
- Archive location history
- Analytics
- Audit trails

**Flow:**
- Driver update → Redis (immediate)
- Driver update → Kafka → PostgreSQL (async batch)

**Key Learning**: Traditional databases don't handle 2D spatial queries well at scale. Use specialized geospatial tools (Redis Geo for real-time, PostGIS for persistence).

---

### 2. Reducing Location Update Frequency

#### The Problem Explained

Even with Redis Geo handling 2M writes/second, we're still wasting resources. Think about it:

- Driver is parked, eating lunch → Still sending location every 5 seconds (unnecessary!)
- Driver stuck in traffic → Barely moving but sending updates (wasteful!)
- Driver speeding on highway → Location changes rapidly (needs frequent updates!)

**Why this matters**:
- **Network bandwidth**: Millions of unnecessary GPS pings
- **Battery drain**: Driver's phone constantly sending data
- **Server load**: Processing updates that don't provide new information
- **Cost**: Paying for data transfer and processing

#### Solution: Smart Client-Side Logic

Make the driver's phone "smart" - it decides when to send updates based on what's happening.

**Adaptive Update Strategy:**

**Scenario 1: Driver is STATIONARY**
- 🚗 (parked, stopped at light)
- → Update every 30 seconds
- → Reduces updates by 6x

**Scenario 2: Driver MOVING SLOWLY**
- 🚗 (city traffic, < 20 mph)
- → Update every 10 seconds
- → Reduces updates by 2x

**Scenario 3: Driver MOVING FAST**
- 🚗 (highway, > 40 mph)
- → Update every 3 seconds
- → Location changes significantly

**Scenario 4: SIGNIFICANT CHANGE**
- 🚗 (turned corner, changed direction)
- → Update IMMEDIATELY
- → Accuracy is critical

**How the Phone Detects This:**

**1. ACCELEROMETER**
- Detects: Movement, vibration, turns
- Logic: No movement = reduce frequency; High acceleration = increase

**2. GPS ACCURACY**
- Detects: Location precision
- Logic: Poor signal = wait for better fix; Good signal = send update

**3. GYROSCOPE**
- Detects: Direction changes, turns
- Logic: Sharp turn = immediate update

**4. SPEED CALCULATION**
- Detects: How fast driver is moving
- Logic: Fast = frequent updates; Slow = infrequent updates

**5. BATTERY LEVEL**
- Detects: Phone battery status
- Logic: Low battery = reduce frequency (keep driver online longer)

**Example Algorithm in Driver App:**

```javascript
function determineUpdateInterval():
  speed = getGPSSpeed()
  movement = getAccelerometerData()
  batteryLevel = getBatteryLevel()

  if movement < threshold:
    return 30 seconds  // Stationary

  if batteryLevel < 20%:
    return 15 seconds  // Save battery

  if speed > 40 mph:
    return 3 seconds   // Fast-moving

  if speed > 10 mph:
    return 10 seconds  // City driving

  return 5 seconds     // Default

// Also send immediate update if:
// - Direction changed > 45 degrees
// - Distance moved > 100 meters
// - Driver accepted/completed ride
```

**Real-World Impact**:
- BEFORE: 10M drivers × 12 updates/min = 120M updates/min = 2M/sec
- AFTER: Adaptive intervals reduce to ~830K/sec
- **60% reduction** in server load, costs, and battery drain
- Same accuracy for matching

**Key Learning**: Don't neglect client-side logic! Smart update strategies can reduce server load by 60%+ while maintaining accuracy. Use device sensors (accelerometer, GPS, gyroscope) to determine optimal update frequency.

---

### 3. Prevent Double-Booking Drivers

#### The Problem Explained

Imagine this scenario:
- **Rider A** requests a ride → System finds Driver Bob
- **Rider B** requests a ride at same time → System also finds Driver Bob (he's closest!)
- Both riders think Bob is coming to pick them up
- One rider will be disappointed and waiting forever

Or worse:
- System sends ride request to Driver Bob
- Bob is reviewing the request (hasn't accepted yet)
- System sends ANOTHER ride request to Bob
- Bob accepts both by accident → chaos!

This is called a **race condition** - two processes "racing" to use the same resource. Without locking, both requests check Bob's availability before either reserves him.

#### Solution: Distributed Locking

Think of it like a bathroom lock - only one person can lock the door at a time.

**Using Redis SETNX (SET if Not eXists):**

**How Distributed Locking Works:**

**Step 1: Try to acquire lock**
```
SETNX driver:bob:lock "rider_A" EX 10
```
Translation: "Set driver bob's lock to 'rider_A' ONLY if no lock exists, and make it expire in 10 seconds"

**Step 2: Check result**
- If returns 1 → Lock acquired! ✓ (You're first, Bob is yours)
- If returns 0 → Lock exists ✗ (Someone else got Bob first)

**Step 3: Use the resource**
- Send notification to Bob
- Wait for accept/decline

**Step 4: Release lock**
```
DEL driver:bob:lock
```
(Make Bob available again)

**Step 5: Auto-expiration (safety net)**
- If service crashes, lock expires in 10s
- Bob becomes available automatically

**Real Flow with Locking:**

**Two Simultaneous Requests (With Locking):**

**Rider A Thread:**
1. Query drivers → Result: Bob
2. `SETNX bob:lock` → ✓ Success! (returns 1, Lock = "A")
3. Send notification to Bob
4. Bob accepts
5. `DEL bob:lock` → Released, Bob available again

**Redis:**
- Manages locks atomically
- Ensures only one rider gets each driver

**Rider B Thread:**
1. Query drivers → Result: Bob
2. `SETNX bob:lock` → ✗ Failed! (returns 0, Lock exists)
3. Try next driver (Alice)
4. `SETNX alice:lock` → ✓ Success!
5. Send notification to Alice

**Result:** ✓ A gets Bob, B gets Alice - No conflicts!

**Complete Algorithm:**

```javascript
function matchDriver(rideRequest):
  nearbyDrivers = findNearbyDrivers()
  sortByProximityAndRating(nearbyDrivers)

  for each driver in nearbyDrivers:
    // Try to lock this driver
    key = "driver:" + driver.id + ":lock"
    lockAcquired = SETNX(
      key,
      rideRequest.id,
      expiry=10 seconds
    )

    if lockAcquired:
      // We got the lock!
      sendNotification(driver, rideRequest)
      return driver

    // Lock failed, try next driver
    continue

  return null  // No drivers available

// When driver responds:
function onDriverResponse(driver, accepted):
  key = "driver:" + driver.id + ":lock"
  DEL(key)  // Release the lock

  if accepted:
    assignRide(driver)
  else:
    matchDriver(rideRequest)  // Try next
```

**Why 10 Second Expiry?**

- **Without expiry**: Lock remains forever if service crashes → Driver stuck → Manual intervention needed
- **With 10s expiry**: Lock auto-released → Driver becomes available again → System self-heals!
- **Why 10 seconds?**: Long enough for driver to read notification (typical 3-7s), short enough for quick recovery if crash

**Alternative: Database Optimistic Locking**

Use a version field in the database:

**Driver table:**
```
id, name, status, version
123, Bob, available, 5
```

**Update with version check:**
```sql
UPDATE drivers
SET status = 'busy',
    version = version + 1
WHERE id = 123
  AND version = 5
  AND status = 'available'
```

**Results:**
- If rows_affected = 1: Success! ✓
- If rows_affected = 0: Conflict! ✗ (Someone else updated it)

**Pros:** Works with existing DB
**Cons:** Slower than Redis, DB load

**Key Learning**: Use distributed locks (Redis SETNX) for strong consistency across services. Similar pattern to Ticketmaster seat reservation - prevent double-booking with atomic operations. Lock expiry prevents deadlocks if service crashes.

---

### 4. Ensure No Dropped Requests During Peak Demand

#### Why This Matters

Imagine New Year's Eve at midnight - thousands requesting rides simultaneously. System receives 100,000 requests in minutes but can only process 10,000/minute.

**Without proper handling**: 90,000 requests dropped = angry customers + lost revenue

**Real-world impact**: Lost revenue, poor UX, bad PR during high-traffic events, no audit trail.

#### Solution: Message Queue (Kafka)

A message queue is like a line at a coffee shop - requests wait their turn instead of overwhelming the system.

```
┌─────────────────────────────────────────────┐
│ Message Queue Architecture                  │
├─────────────────────────────────────────────┤
│ ┌──────────┐    ┌────────────┐    ┌──────┐ │
│ │ Ride     │───→│   Kafka    │───→│Match │ │
│ │ Service  │    │   Queue    │    │Svc   │ │
│ │          │    │ ┌────────┐ │    │      │ │
│ │ Fast     │    │ │Msg 1-N │ │    │Process│ │
│ │ write    │    │ │100,000+│ │    │at own│ │
│ └──────────┘    │ │Durable │ │    │pace  │ │
│                 │ └────────┘ │    └──────┘ │
│                 │ Persisted  │             │ │
│                 │ to disk    │             │ │
│                 └────────────┘             │ │
└─────────────────────────────────────────────┘
```

**Benefits**:
1. **Durability**: Messages survive crashes - persisted to disk
2. **Load Leveling**: Smooth traffic spikes - queue absorbs bursts
3. **Retry Logic**: Auto-retry on failures - transient errors handled
4. **Decoupling**: Services scale independently

**Flow**:
1. Ride Service → Kafka (milliseconds, returns immediately)
2. Kafka persists to disk
3. Matching Service consumes at sustainable pace
4. If crash occurs → Resume from last checkpoint
5. All requests eventually processed ✓

**Key Learning**: Kafka provides durability, retry logic, and load leveling. Essential for high-volume critical workflows. Handles millions of messages/sec with ordering guarantees.

---

### 5. Handle Driver Timeout

#### Why This Matters

Driver drops phone in passenger seat and takes a break. Ride request sent but no response. Without timeout handling, rider waits forever.

**Matching Workflow with Timeouts:**

**1. Get ranked list of drivers**
- [Driver A, Driver B, Driver C]

**2. Try Driver A:**
- → Acquire lock
- → Send notification
- → Start 10-second timer
- → Wait for response...

**Option 1: Driver accepts (< 10s)**
- → Release lock
- → Assign ride ✓
- → END workflow

**Option 2: Driver declines (< 10s)**
- → Release lock
- → Continue to Driver B

**Option 3: No response (timeout)**
- → Timer expires after 10s
- → Auto-release lock
- → Continue to Driver B

**3. Try Driver B** (repeat process)

**4. Try Driver C** (repeat process)

**5. If all drivers exhausted:**
- → No drivers available
- → Notify rider

**What Makes Temporal Special:**

**1. STATE PERSISTENCE**
- Workflow state saved after each step
- Service crashes? Resume from last step
- No lost progress

**2. RELIABLE TIMERS**
- 10-second timeout? Guaranteed
- Works across restarts
- Never loses track of time

**3. AUTOMATIC RETRIES**
- Network error? Auto-retry
- Configurable backoff
- Exponential retry logic built-in

**4. VISIBILITY**
- See workflow history
- Debug stuck workflows
- Audit trail for compliance

**Code Example (Temporal Workflow Pseudo-code):**

```javascript
workflow MatchDriver(rideId):
  drivers = findNearbyDrivers(rideId)

  for driver in drivers:
    // Acquire lock (activity)
    lockAcquired = acquireLock(driver.id)
    if not lockAcquired:
      continue

    // Send notification (activity)
    sendNotification(driver, rideId)

    // Wait with timeout
    response = await_signal(
      signal_name="driver_response",
      timeout=10_seconds
    )

    // Release lock (activity)
    releaseLock(driver.id)

    if response == "accept":
      assignRide(driver, rideId)
      return SUCCESS

    // Declined or timeout, try next
    continue

  return NO_DRIVERS_AVAILABLE

// Activities are actual work:
activity acquireLock(driverId):
  return redis.setnx(...)

activity sendNotification(driver, ride):
  apns.send(driver.deviceToken, ...)
```

**Why Not Just Use a Simple Loop?**

**PROBLEM: Service crashes mid-loop**

**Regular Code:**
```javascript
for driver in drivers:
  notify(driver)
  wait(10s)  ← CRASH HERE! 💥
// Lost state, can't resume
// Which drivers already notified?
// Start over? Skip? Unknown!
```

**Temporal Workflow:**
```javascript
for driver in drivers:
  notify(driver)  ← Saved to DB
  wait(10s)       ← CRASH HERE! 💥
// Temporal auto-restarts workflow
// Resumes from exact point
// Knows we're waiting for driver #2
// Continues seamlessly ✓
```

**Real-World Flow:**

| Time | Workflow State | What Happened |
|------|----------------|---------------|
| 0:00 | Start workflow | Rider requests |
| 0:01 | Lock Driver A | SETNX success |
| 0:01 | Notify Driver A | Push notification |
| 0:01 | Wait (timer: 10s) | Temporal timer |
| | [Service crashes! 💥] | |
| 0:05 | [Service restarts] | |
| 0:05 | Wait (timer: 5s left) | Temporal resumes |
| 0:11 | Timeout! | Timer expired |
| 0:11 | Release lock A | DEL lock |
| 0:11 | Lock Driver B | SETNX success |
| 0:11 | Notify Driver B | Push notification |
| 0:11 | Wait (timer: 10s) | Temporal timer |
| 0:15 | Signal: "accept" | Driver B accepts |
| 0:15 | Release lock B | DEL lock |
| 0:15 | Assign ride | Success! ✓ |
| 0:15 | End workflow | Complete |

**Key Learning**: Human-in-the-loop processes need durable execution. Temporal persists workflow state so crashes don't lose progress. Uber created Cadence (open-sourced, evolved into Temporal) specifically for this use case. Essential for multi-step processes with waits/timeouts.

---

### 6. Further Scaling Strategies

As Uber grows globally, additional optimizations become necessary.

**Horizontal Scaling Techniques:**

**1. CDN for Static Assets**
- App images, maps tiles
- Reduced latency (edge locations)
- Lower origin server load

**2. Database Read Replicas**
- Master: writes
- Replicas: reads (driver profiles, etc)
- Distribute read load across regions

**3. Redis Caching**
- Frequent queries (driver ratings)
- Fare estimates (cache similar routes)
- Reduces DB load by 70%+

**4. Geographic Sharding**
- US-East drivers → US-East DB
- EU drivers → EU DB
- Reduces cross-region latency

**5. Service Mesh (Istio/Linkerd)**
- Load balancing between services
- Circuit breakers
- Observability

**6. Auto-scaling**
- Scale matching service during peaks
- K8s HPA based on queue depth
- Save costs during quiet hours

**7. Multi-Region Deployment**
- Active-active in multiple regions
- Disaster recovery
- Lower latency for global users

**Geographic Sharding Example:**

**San Francisco Shard:**
- San Francisco Requests
  - ↓ SF Redis Geo
  - ↓ SF PostgreSQL
  - ↓ SF Matching Service
  - ↓ Only SF drivers

**New York Shard:**
- New York Requests
  - ↓ NY Redis Geo
  - ↓ NY PostgreSQL
  - ↓ NY Matching Service
  - ↓ Only NY drivers

**Benefits:**
- Lower latency (data closer to users)
- Isolated failures (SF down ≠ NY down)
- Easier to scale per region
- Comply with data residency laws

**Key Learning**: Use CDN, caching, read replicas, and geographic sharding to scale globally. Auto-scale services based on demand. Multi-region deployment for high availability.

---

## Final Architecture

```
┌──────────┐  ┌──────────┐
│  Rider   │  │  Driver  │
│  Client  │  │  Client  │
└────┬─────┘  └────┬─────┘
     │             │
     └──────┬──────┘
            ↓
     ┌────────────┐
     │   API      │
     │  Gateway   │
     └──────┬─────┘
            │
    ┌───────┴────────┐
    ↓                ↓
┌─────────┐    ┌──────────┐
│  Ride   │    │ Location │
│ Service │    │ Service  │
└────┬────┘    └────┬─────┘
     │              │
     ↓              ↓
┌─────────┐    ┌──────────┐
│ Kafka   │    │ Redis    │
│ Queue   │    │ Geo      │
└────┬────┘    └──────────┘
     │
     ↓
┌──────────┐    ┌──────────┐
│ Matching │───>│  Redis   │
│ Service  │    │  Lock    │
│(Temporal)│    └──────────┘
└────┬─────┘
     │
     ↓
┌──────────┐
│Notification│
│ Service  │
└──────────┘
```

**Key Components**:
- **Redis Geo**: Location storage & proximity search
- **Kafka**: Durable ride request queue
- **Redis Lock**: Prevent driver double-booking
- **Temporal**: Durable execution for matching workflow
- **APNs/FCM**: Push notifications to drivers

---

## Key Takeaways

1. **Sequential approach**: Build functional requirements one-by-one
2. **Geospatial indexing**: Critical for location-based queries (QuadTree, Redis Geo, PostGIS)
3. **Client-side optimization**: Reduce server load with smart update logic
4. **Distributed locking**: Ensure consistency (Redis SETNX)
5. **Message queues**: Durability for critical workflows (Kafka/SQS)
6. **Durable execution**: Handle multi-step human workflows (Temporal)
7. **Security**: Never trust client data (use sessions/JWT, server timestamps)


