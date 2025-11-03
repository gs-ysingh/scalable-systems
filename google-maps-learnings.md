# Google Maps System Design - Key Learnings

## Table of Contents
1. [Overview](#overview)
2. [Core Requirements](#core-requirements)
3. [Scale & Capacity](#scale--capacity)
4. [Dijkstra's Algorithm Basics](#dijkstras-algorithm-basics)
5. [Contraction Hierarchies](#contraction-hierarchies)
6. [Database Design](#database-design)
7. [Partitioning Strategy](#partitioning-strategy)
8. [Real-time ETA Updates](#real-time-eta-updates)
9. [Complete System Architecture](#complete-system-architecture)

---

## Overview

Google Maps is a navigation service that provides:
- Route finding between locations
- Estimated Time of Arrival (ETA)
- Real-time traffic updates

**Key Challenge**: Finding optimal routes across billions of road segments while maintaining low latency.

---

## Core Requirements

### Functional Requirements
1. **Route Finding**: Calculate the quickest route between two arbitrary locations
2. **ETA Calculation**: Provide accurate time estimates for routes
3. **Dynamic Updates**: Incorporate real-time traffic, weather, and road conditions into ETAs

### Trade-offs
- Balance between finding the *best* route vs. providing *fast* response times
- Not always the absolute quickest route, but a very good route delivered quickly

---

## Scale & Capacity

### Graph Size (USA alone)
- **16 million** intersections
- **200 million** places/addresses
- **~50 billion** nodes globally (estimated)
- Countless edges (roads) connecting them

### Data Requirements
Each node/edge requires metadata:
- Latitude & Longitude
- Unique ID
- Name
- Travel time (for edges)
- Road type, speed limits, etc.

**Key Insight**: The entire graph cannot fit on a single partition - requires massive distributed storage.

---

## Dijkstra's Algorithm Basics

### How It Works

Dijkstra's algorithm finds the shortest path by maintaining three categories of nodes:

```
┌──────────────┬─────────────────────────────────────────────────┐
│ VISITED      │ Shortest path from start is CONFIRMED          │
├──────────────┼─────────────────────────────────────────────────┤
│ DISCOVERED   │ Node reached, but might find shorter path      │
├──────────────┼─────────────────────────────────────────────────┤
│ UNEXPLORED   │ Not encountered yet                             │
└──────────────┴─────────────────────────────────────────────────┘
```

**Core Principle**: Always process the closest unvisited node first (greedy approach).

### Example Graph

```
        4           2
    A ─────> B ─────> E (Goal)
    │        │        ↑
   1│       3│        │1
    ↓        ↓        │
    C ─────> D ───────┘
        2           5
```

**Edge weights represent travel time (minutes)**

### Step-by-Step Execution

**Goal**: Find shortest path from A to E

#### **Initial State**
```
Visited: []
Discovered: [A(0)]  ← distance from start
Unexplored: [B, C, D, E]
```

---

#### **Iteration 1**: Process A (distance = 0)
```
Current: A (0)
Neighbors: B (via A: 0+4=4), C (via A: 0+1=1)

Visited: [A✓]
Discovered: [C(1), B(4)]  ← sorted by distance
Unexplored: [D, E]
```
**Action**: Mark A as visited, add its neighbors to discovered set

---

#### **Iteration 2**: Process C (distance = 1)
```
Current: C (1)
Neighbors: D (via C: 1+2=3)

Visited: [A✓, C✓]
Discovered: [D(3), B(4)]
Unexplored: [E]
```
**Why C first?** It's the closest unvisited node (distance 1 < 4)

---

#### **Iteration 3**: Process D (distance = 3)
```
Current: D (3)
Neighbors: E (via D: 3+5=8), B (via D: 3+3=6)

Visited: [A✓, C✓, D✓]
Discovered: [B(4), E(8)]  ← B stays at 4 (4 < 6, don't update)
Unexplored: []
```
**Key insight**: We found a path to B via D (distance 6), but already have a shorter path (distance 4), so we keep the shorter one.

---

#### **Iteration 4**: Process B (distance = 4)
```
Current: B (4)
Neighbors: E (via B: 4+2=6)

Visited: [A✓, C✓, D✓, B✓]
Discovered: [E(6)]  ← updated! 6 < 8
Unexplored: []
```
**Update**: Found shorter path to E through B (6 vs 8), update distance!

---

#### **Iteration 5**: Process E (distance = 6) - GOAL REACHED!
```
Current: E (6)

Visited: [A✓, C✓, D✓, B✓, E✓]
Discovered: []
Unexplored: []
```

**Final Result**: Shortest path A→B→E with total distance **6 minutes**
- Path: A → B (4 min) → E (2 min) = 6 min
- Alternative path A→C→D→E would be 1+2+5 = 8 min ❌

### Time Complexity
- Depends on implementation (array, priority queue, etc.)
- **O(V² )** or **O((V + E) log V)** with priority queue
- Where V = vertices, E = edges

### The Problem
For long distances (Delhi to Mumbai), we'd need to consider:
- Millions of nodes
- Billions of edge combinations
- **Not feasible** for real-time queries!

---

## Contraction Hierarchies

### The Solution: Reduce Search Space

Instead of searching the entire graph, create **contracted graphs** that only include important roads.

### Two Key Techniques

#### 1. Remove Unimportant Edges

```
Before:           After:
A ─1─> B ─1─> C   A ─1─> B ─1─> C
  └─────5──────┘   (removed edge A->C)
```

If A→B→C is faster than A→C, remove the A→C edge.

#### 2. Create Shortcut Edges

```
Before:                    After:
A ─2─> X ─2─> Y ─2─> B    A ────6────> B (shortcut)
                          (removed X, Y)
```

Replace sequences of less important roads with a single "shortcut" edge.

### Multiple Levels of Sparsity

```
┌────────────────────────────────────────┐
│ LEVEL 3: Super Sparse (Major highways)│
│ - Interstate highways only             │
│ - Cross-country routes                 │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ LEVEL 2: Sparse (State highways)       │
│ - Major highways + state routes        │
│ - Regional travel                      │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ LEVEL 1: Full Graph (All roads)        │
│ - Every street, intersection           │
│ - Local navigation                     │
└────────────────────────────────────────┘
```

### Why Multiple Levels?

- **Short distances** (< 10 min): Use full graph
- **Medium distances** (10 min - 2 hrs): Use sparse graph
- **Long distances** (> 2 hrs): Use super sparse graph

This dramatically reduces search space while maintaining accuracy.

### Route Finding with Contraction Hierarchies

Let's see how to find a route from **Home** to **Airport** using the hierarchical approach.

#### **The Full Graph (Level 1)**
```
       Local Roads (many small streets)
       
Home ──2─┐  ┌──3── Mall ──4─┐  ┌──2── Airport
         │  │                │  │
      Highway               Highway
      Entrance              Exit
      [HE1]                 [HX1]
         └──5───> I-95 ──8──>┘
                (Highway)
```

#### **The Sparse Graph (Level 2 - Only Major Routes)**
```
Just the highway system:

[HE1] ─────13────> [HX1]
Highway              Highway
Entrance             Exit
```

---

### **Step-by-Step Route Finding**

#### **Phase 1: Connect Start to Sparse Graph**

Run mini-Dijkstra from **Home** to find nearest highway entrances:

```
Home ──2──> [HE1]  ✓ (2 minutes)
Home ──2──> Mall ──1──> [HE2]  (3 minutes, alternate entrance)
```

**Best entry point**: HE1 at 2 minutes

---

#### **Phase 2: Connect End to Sparse Graph**

Run mini-Dijkstra from **Airport** to find nearest highway exits:

```
Airport <──2── [HX1]  ✓ (2 minutes)
Airport <──4── [HX2]  (4 minutes, alternate exit)
```

**Best exit point**: HX1 at 2 minutes

---

#### **Phase 3: Search Sparse Graph**

Now run Dijkstra on the sparse (highway) graph:

```
Possible routes between HE1 and HX1:

Route A: [HE1] ─13─> [HX1]  (13 minutes via I-95)
Route B: [HE1] ─8─> [Hub] ─7─> [HX1]  (15 minutes via I-87)
Route C: [HE1] ─10─> [West] ─12─> [HX1]  (22 minutes, scenic)
```

**Best highway route**: Route A at 13 minutes

---

#### **Phase 4: Calculate Total Time**

```
┌──────────────┬──────────┬─────────────┬───────────┬────────┐
│ Entry Route  │  Entry   │   Highway   │   Exit    │  Total │
│              │   Time   │   Segment   │   Time    │  Time  │
├──────────────┼──────────┼─────────────┼───────────┼────────┤
│ Home → HE1   │   2 min  │  HE1 → HX1  │   2 min   │ 17 min │✓
│ Home → HE2   │   3 min  │  HE2 → HX1  │   2 min   │ 19 min │
│ Home → HE1   │   2 min  │  HE1 → HX2  │   4 min   │ 20 min │
└──────────────┴──────────┴─────────────┴───────────┴────────┘
```

**Optimal Route**: Home → HE1 → I-95 → HX1 → Airport = **17 minutes**

---

### **Why This Works**

#### **Without Contraction Hierarchies**:
```
Would need to explore ALL local roads:
- 1000s of residential streets
- Every traffic light
- Shopping mall parking lots
- Side streets and alleys
= Millions of possible paths! ❌
```

#### **With Contraction Hierarchies**:
```
Only explore:
- ~10 local roads to find highway entrance (small search)
- ~5 highway routes (sparse graph search)
- ~10 local roads from highway exit (small search)
= Total ~25 roads to consider! ✓
```

**Speed-up**: From exploring millions of edges to just dozens!

---

### **Real-World Example: New Delhi to Bengaluru**

```
Level 1 (Full):     Every street in Delhi + Haryana + Karnataka + Bengaluru
                    (~12 million road segments)

Level 2 (Sparse):   NH-44, NH-48, major state highways
                    (~600 highway segments)

Level 3 (Super):    Just NH-44 (the main corridor Delhi-Bengaluru)
                    (~80 segments)
```

**Search Process**:
1. Find NH-44 entrance near you in Delhi (search ~1200 local roads)
2. Follow NH-44 through Haryana → UP → MP → Karnataka (search ~80 highway segments)
3. Find route from NH-44 exit to destination in Bengaluru (search ~1000 local roads)

**Total**: ~2300 roads explored instead of 12 million! 🚀

**Distance**: ~2,150 km journey made searchable in milliseconds!

---

## Database Design

### 1. Full Graph Storage: Neo4j (Graph Database)

**Why Neo4j?**
- **Native graph database**: Uses pointers on disk for edges
- **O(1) traversal** between nodes (vs O(log N) with SQL indexes)
- Performance doesn't degrade as dataset grows

**Alternative (Not Recommended)**: SQL with many-to-many relationships
- Uses indexes for edge lookups
- Binary search = O(log N) per hop
- Gets slower as data grows

### 2. Sparse Graphs: In-Memory Cache

**Why In-Memory?**
- Accessed frequently (most queries use sparse graphs)
- Smaller size makes caching feasible
- Dramatically faster read speeds
- Backup copy still in Neo4j

### Database Architecture

```
┌─────────────────────────────────────────┐
│         Full Graph (Neo4j)              │
│  - All roads, intersections, places     │
│  - Sharded by geohash                   │
│  - Persistent storage                   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│      Sparse Graph (In-Memory)           │
│  - Major highways, important routes     │
│  - Cached for fast access               │
│  - Backup in Neo4j                      │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│   Super Sparse Graph (In-Memory)        │
│  - Interstate highways only             │
│  - Possibly single partition            │
└─────────────────────────────────────────┘
```

---

## Partitioning Strategy

### Geohash Partitioning

**Geohash**: A location encoding where nearby points have similar IDs

```
┌──────┬──────┬──────┐
│ 9q5  │ 9q7  │ 9qd  │  Each cell = geohash region
├──────┼──────┼──────┤
│ 9q4  │ 9q6  │ 9qc  │  Similar geohashes = nearby
├──────┼──────┼──────┤
│ 9q1  │ 9q3  │ 9q9  │  locations
└──────┴──────┴──────┘
```

**Benefits**:
- Dijkstra searches stay mostly within one partition
- Starting point determines primary partition
- Efficient for local queries

### Handling Hot Partitions

**Problem**: Mumbai gets more traffic than rural Rajasthan

**Solution 1 - Dynamic Sizing**:
```
┌─────┐  Mumbai (small dense area, many users)
│ MUM │  
└─────┘  

┌─────────────┐  Rajasthan (large sparse area, fewer users)
│  Rajasthan  │
└─────────────┘
```

**Solution 2 - More Replicas**:
- Mumbai partition: 20 replicas
- Rajasthan partition: 2 replicas
- Since mostly read-only, replication works well

**Trade-off**: More replicas = harder to update with traffic data

---

## Real-time ETA Updates

### The Complete Flow

#### Step 1: Calculate Driver Speed

```
GPS Signal 1      GPS Signal 2
    │                 │
    ├─ Lat/Long       ├─ Lat/Long
    ├─ Timestamp      ├─ Timestamp
    │                 │
    └────── Distance ──┘
           ─────────────
            Time Δ
                = Speed
```

**Note**: Phones can't directly measure speed; need to calculate from GPS displacement over time.

#### Step 2: Identify Which Road (Hidden Markov Model)

```
GPS Signal 1: (40.7128, -74.0060)  ← Which road?
                ↓
         ┌──────┴──────┐
    Road A (80%)   Road B (20%)  ← Probability
         │              │
GPS Signal 2: (40.7150, -74.0050)
         │              │
    Road A (95%)   Road B (5%)   ← More confident now
```

**Hidden Markov Model Components**:
- **Emission Probability**: Likelihood of GPS signal given road position
- **Transition Probability**: Likelihood of moving from one road to another
- **Viterbi Algorithm**: Finds most likely sequence of roads

**Implementation**:

**Why Kafka?**
- **Distributed Message Queue**: Handles millions of GPS signals per second from drivers
- **Partitioning**: Routes GPS signals from same geographic area to same Flink node
- **Durability**: Stores GPS data temporarily in case Flink nodes crash
- **Decoupling**: Separates GPS producers (phones) from consumers (processing)

**Why Flink?**
- **Stateful Stream Processing**: Remembers previous GPS points to run Viterbi algorithm
- **Low Latency**: Processes GPS signals in real-time (milliseconds)
- **Exactly-Once Processing**: Ensures each GPS signal is processed exactly once
- **Scalable**: Can add more Flink nodes as traffic increases

**Data Flow**:
- Partition by geohash to maintain state (same driver → same Flink node)
- GPS signals → Road ID + Speed

#### Step 3: Calculate Average Road Speed

**Challenge**: Need average of all drivers, not just one

**Solution**: Flink with Efficient Data Structure

```
Road ID: 12
┌────────────────────────────────────┐
│ Linked List: [22] → [25] → [28] → [20] mph  │
│ Entries: 4                         │
│ Total: 95 mph                      │
│ Average: 95 / 4 = 23.75 mph        │
└────────────────────────────────────┘
```

**Operations**:
- **Add new speed**: Append to list, update total & count → O(1)
- **Remove old speed** (> 5 min): Remove from front → O(1)
- **Calculate average**: Total / Entries → O(1)

**Configuration**: Aggregate every 5-10 minutes

**Why Kafka + Flink Again?**

**Kafka's Role**:
- Receives (Road ID, Speed) pairs from previous stage
- Partitions by Road ID (all speeds for Road 12 → same partition)
- Acts as buffer between stages (decoupling)

**Flink's Role**:
- Maintains **state** for each road (linked list, counters)
- **Windowing**: Groups speeds in 5-minute time windows
- **Aggregation**: Computes rolling average efficiently
- Partition alignment with Kafka ensures all data for one road goes to same Flink node

**Why Partition by Road ID?**
- Road 12's speeds from 1000 drivers → same Flink node
- That node maintains state for Road 12 only
- Parallel processing: Different roads processed simultaneously on different nodes

#### Step 4: Update Graph Edges

**Simple case**: Update base edge weight

```
Road 12: 10 min → 15 min (traffic!)
```

**Complex case**: Shortcut edges need updating

```
Before Traffic:
A ─1─> X ─1─> Y ─1─> B
Shortcut: A ─3─> B

After Traffic on X-Y:
A ─1─> X ─6─> Y ─1─> B
Shortcut: A ─8─> B  ← Needs update!

Better route exists:
A ─2─> M ─2─> N ─2─> B
New Shortcut: A ─6─> B (via M, N)
```

#### Step 5: Bubble Up Dependencies

**Problem**: Shortcut edges depend on base edges. When base edge changes, shortcuts may need recalculation.

**Dependency Tracking**:

```mysql
shortcut_dependencies table:
┌─────────────┬──────────────┐
│ shortcut_id │ depends_on   │
├─────────────┼──────────────┤
│ 46          │ 12           │
│ 46          │ 13           │
│ 46          │ 14           │
│ 197         │ 46           │
│ 197         │ 15           │
└─────────────┴──────────────┘
```

**Why MySQL?**

**Use Case**: Store which shortcut edges depend on which base edges

**Why MySQL is Perfect Here**:
- **Relational Data**: Natural fit for dependency relationships
- **ACID Compliance**: Ensures dependency integrity (no partial updates)
- **Infrequent Writes**: Dependencies only change when recomputing shortcuts (rare)
- **Simple Queries**: Just lookup by depends_on column (indexed)
- **Change Data Capture (CDC)**: Built-in support to stream changes to Kafka

**Alternatives Considered**:
- ❌ **NoSQL** (Cassandra/MongoDB): Overkill, no ACID needed here
- ❌ **In-Memory** (Redis): Data must persist, not just cached
- ✓ **MySQL**: Simple, reliable, perfect for this use case

**When to Recalculate Shortcut**:
- Only if base edge changes significantly (threshold based)
- If shortcut ETA increases by > X%
- Run Dijkstra on full graph to find new shortcut path

**Complete Data Flow**:
```
1. MySQL (stores dependencies)
    ↓ 
2. CDC (Change Data Capture - detects INSERT/UPDATE/DELETE)
    ↓
3. Kafka (buffers change events)
    ↓
4. Flink (enriches speed updates with dependency info)
    ↓
5. Update: Full Graph, Sparse Graph, Super Sparse Graph
```

**How CDC Works**:
- MySQL writes to binary log (binlog) for every change
- CDC tool (Debezium) reads binlog in real-time
- Publishes changes as events to Kafka
- Flink subscribes and knows "Road 12 is part of Shortcut 46"
- When Road 12 speed changes, Flink also updates Shortcut 46

### Data Flow Diagram

```
┌─────────┐
│ Driver  │ GPS signals
└────┬────┘
     │
     ↓
┌─────────────────────────────────────┐
│  Hidden Markov Model Service        │
│  (Kafka + Flink, partition by geo)  │
│  Output: Road ID + Speed             │
└────────┬────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────┐
│  Speed Aggregation                  │
│  (Flink, partition by Road ID)      │
│  Output: Avg speed per road         │
└────────┬────────────────────────────┘
         │
         ↓
    ┌────┴─────┐
    │  Flink   │ checks dependencies
    │  (CDC)   │ 
    └────┬─────┘
         │
    ┌────┴──────────────────────┐
    │                           │
    ↓                           ↓
┌────────────┐         ┌──────────────────┐
│ Full Graph │         │ Shortcut Deps DB │
│  (Neo4j)   │←─ CDC ──│    (MySQL)       │
└─────┬──────┘         └──────────────────┘
      │
      ├─→ Sparse Graph (in-memory)
      │
      └─→ Super Sparse Graph (in-memory)
```

---

## Complete System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          CLIENT (Driver)                          │
└───────────┬──────────────────────────────────┬───────────────────┘
            │                                  │
            │ Route Request                    │ GPS Pings
            ↓                                  ↓
┌────────────────────────┐      ┌────────────────────────────────┐
│   Route Service        │      │  Traffic Update Service        │
│   - Calculate route    │      │  - Receive GPS signals         │
│   - Return ETA         │      └──────────┬─────────────────────┘
└───────┬────────────────┘                 │
        │                                  ↓
        │                         ┌─────────────────┐
        │                         │     Kafka       │
        │                         └────────┬────────┘
        │                                  │
        │                                  ↓
        │                    ┌──────────────────────────────┐
        │                    │  Hidden Markov Model (Flink) │
        │                    │  - GPS → Road ID + Speed     │
        │                    │  - Partition by geohash      │
        │                    └──────────┬───────────────────┘
        │                               │
        │                               ↓
        │                    ┌─────────────────┐
        │                    │     Kafka       │
        │                    └────────┬────────┘
        │                             │
        │                             ↓
        │              ┌────────────────────────────────┐
        │              │  Speed Aggregator (Flink)      │
        │              │  - Partition by Road ID        │
        │              │  - 5-min rolling average       │
        │              │  - Check shortcut dependencies │
        │              └──────┬─────────────────────────┘
        │                     │
        │         ┌───────────┼───────────────┐
        │         │           │               │
        ↓         ↓           ↓               ↓
┌────────────────────┐  ┌──────────┐  ┌──────────────────┐
│  Super Sparse Map  │  │  Sparse  │  │   Full Graph     │
│   (In-Memory)      │  │   Map    │  │    (Neo4j)       │
│  - Interstate only │  │(In-Mem)  │  │  - All roads     │
│  - Long distance   │  │- Highways│  │  - Partitioned   │
└────────────────────┘  └──────────┘  └────────┬─────────┘
                                               │
                                               │ Recalculate shortcuts
                                               ↓
                                    ┌──────────────────────┐
                                    │ Shortcut Dependencies│
                                    │      (MySQL)         │
                                    │  - Single leader     │
                                    │  - CDC to Kafka      │
                                    └──────────────────────┘
```

### Key Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Route Service | Service Layer | Calculate routes using appropriate graph level |
| Full Graph | Neo4j | Store all roads, sharded by geohash |
| Sparse/Super Sparse | In-Memory Cache | Fast access to important routes |
| Traffic Service | API Gateway | Receive GPS signals from drivers |
| HMM Processing | Kafka + Flink | Convert GPS → Road ID + Speed |
| Speed Aggregation | Flink | Calculate average speeds per road |
| Shortcut Dependencies | MySQL + CDC | Track and update shortcut edges |

---

## Technology Deep Dive: Kafka, Flink, and MySQL

### Why These Three Work Together Perfectly

```
┌────────────────────────────────────────────────────────────┐
│  Stream Processing Pipeline for Real-Time Traffic Updates  │
└────────────────────────────────────────────────────────────┘

  GPS Data          Message Queue      Stream Processor      Database
     ↓                    ↓                   ↓                 ↓
┌─────────┐         ┌─────────┐        ┌─────────┐      ┌──────────┐
│ Phones  │────────>│  KAFKA  │───────>│  FLINK  │─────>│  MYSQL   │
│ (Drivers)│         │         │        │         │      │          │
└─────────┘         └─────────┘        └─────────┘      └──────────┘
  Produces          Distributes         Processes        Stores
  Events            & Buffers           & Maintains      Persistent
                                        State            Data
```

### Kafka: The Distributed Event Streaming Platform

**What It Does**: Acts as a high-throughput, fault-tolerant message queue

**Key Features**:
1. **Partitioning**: Distributes data across multiple brokers
2. **Replication**: Keeps copies of data for fault tolerance
3. **Retention**: Stores messages for configurable time (e.g., 7 days)
4. **Ordering**: Guarantees order within a partition

**In Google Maps Context**:
```
Topic: gps-signals (partition by geohash)
├─ Partition 0: Delhi area GPS signals
├─ Partition 1: Mumbai area GPS signals
├─ Partition 2: Bangalore area GPS signals
└─ Partition N: ...

Topic: road-speeds (partition by road_id)
├─ Partition 0: Road IDs 0-10000
├─ Partition 1: Road IDs 10001-20000
└─ Partition N: ...
```

**Benefits**:
- **Decoupling**: Phones don't talk directly to Flink
- **Buffering**: Can handle traffic spikes (millions of GPS pings/sec)
- **Replay**: Can reprocess data if Flink crashes
- **Scalability**: Add more partitions as traffic grows

---

### Flink: The Stateful Stream Processor

**What It Does**: Processes infinite streams of data with low latency

**Key Features**:
1. **Stateful Processing**: Remembers information across events
2. **Exactly-Once Semantics**: Each event processed exactly once (no duplicates)
3. **Event Time Processing**: Handles out-of-order events correctly
4. **Windowing**: Groups events by time (e.g., 5-minute windows)

**State Management Example**:
```
Flink Node Processing Road 12:

State (in-memory):
├─ Speeds: [22, 25, 28, 20] mph
├─ Count: 4
├─ Sum: 95
└─ Last Update: 14:30:00

New Event Arrives: Road 12, Speed 30 mph, Time 14:31:00
↓
Update State:
├─ Speeds: [25, 28, 20, 30] mph  (dropped old 22)
├─ Count: 4
├─ Sum: 103
└─ Last Update: 14:31:00
↓
Emit: Road 12, Average = 25.75 mph
```

**Why Flink for This Use Case**:
- **Hidden Markov Model**: Needs state (previous GPS points) to infer road
- **Rolling Averages**: Needs state (last 5 min of speeds) to compute average
- **Low Latency**: Processes events in milliseconds
- **Fault Tolerance**: Checkpoints state to survive crashes

**Flink vs Alternatives**:
- ❌ **Spark Streaming**: Micro-batching (seconds latency), not true streaming
- ❌ **Lambda Functions**: Stateless, can't maintain rolling windows
- ✓ **Flink**: True streaming, stateful, exactly-once, millisecond latency

---

### MySQL: The Relational Database

**What It Does**: Stores structured, relational data with ACID guarantees

**Key Features**:
1. **ACID Transactions**: Atomic, Consistent, Isolated, Durable
2. **Indexing**: Fast lookups on indexed columns
3. **Foreign Keys**: Enforces referential integrity
4. **Change Data Capture**: Streams changes in real-time

**Schema for Shortcut Dependencies**:
```sql
CREATE TABLE shortcut_dependencies (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    shortcut_edge_id BIGINT NOT NULL,
    depends_on_edge_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_depends_on (depends_on_edge_id),  -- Fast lookup
    INDEX idx_shortcut (shortcut_edge_id)
);

-- Example query when Road 12 speed changes:
SELECT shortcut_edge_id 
FROM shortcut_dependencies 
WHERE depends_on_edge_id = 12;
-- Returns: [46, 197] in milliseconds
```

**CDC (Change Data Capture) Flow**:
```
1. App inserts new dependency: Shortcut 46 depends on Road 12
   ↓
2. MySQL writes to binary log (binlog)
   ↓
3. Debezium CDC connector reads binlog
   ↓
4. Publishes event to Kafka: {"shortcut": 46, "depends_on": 12, "op": "INSERT"}
   ↓
5. Flink consumes and updates in-memory lookup table
   ↓
6. Now Flink knows: "When Road 12 changes, update Shortcut 46"
```

**Why MySQL Here**:
- **ACID**: Ensures dependency consistency (no partial updates)
- **Simple Queries**: Just lookup by edge_id (indexed, fast)
- **Mature CDC**: Debezium has excellent MySQL support
- **Low Write Volume**: Dependencies change rarely (only on shortcut recompute)

**MySQL vs Alternatives**:
- ❌ **Redis**: No persistence guarantees, data could be lost
- ❌ **Cassandra**: Eventual consistency, no transactions needed
- ❌ **Neo4j**: Overkill for simple lookups, already using for main graph
- ✓ **MySQL**: Simple, reliable, ACID, great CDC support

---

### How They Work Together: Complete Example

**Scenario**: Driver on Road 12 causing traffic jam

```
Step 1: Driver's phone sends GPS
   Phone → Kafka (topic: gps-signals, partition: geohash_xyz)

Step 2: Flink processes GPS with HMM
   Kafka → Flink Node A (partition geohash_xyz)
   Flink: "GPS (28.6, 77.2) + previous points → Road 12, Speed: 15 mph"
   Flink → Kafka (topic: road-speeds, partition: road_12)

Step 3: Flink aggregates speeds for Road 12
   Kafka → Flink Node B (partition road_12)
   Flink: "Road 12 average over 5 min: 15 mph (was 40 mph!)"
   Flink checks dependency table (from MySQL CDC):
      "Road 12 is part of Shortcuts 46 and 197"

Step 4: Flink updates all affected edges
   Flink → Neo4j: Update Road 12 weight (15 mph)
   Flink → Neo4j: Update Shortcut 46 weight (recalculate)
   Flink → Neo4j: Update Shortcut 197 weight (recalculate)

Step 5: Next driver gets updated route
   Driver → Route Service → Neo4j
   Neo4j: "Avoid Road 12 (traffic!), use alternate route"
```

**The Magic**:
- **Kafka** ensures no GPS signal is lost (buffering + replay)
- **Flink** maintains state to compute averages and dependencies
- **MySQL** provides reliable storage with real-time change streaming
- All three together enable **sub-second** traffic updates! 🚀

---

### Data Partitioning Summary

```
┌─────────────────────────────────────────────┐
│ Kafka Topics:                               │
│  - GPS signals: partition by geohash        │
│  - Road speeds: partition by road ID        │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Flink Nodes:                                │
│  - HMM service: partition by geohash        │
│  - Speed aggregator: partition by road ID   │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Neo4j Shards:                               │
│  - Partition by geohash                     │
│  - Hot partitions: more replicas OR smaller │
└─────────────────────────────────────────────┘
```

---

## Key Takeaways

### 1. **Graph Optimization is Critical**
- Raw Dijkstra doesn't scale to billions of nodes
- Contraction hierarchies reduce search space by orders of magnitude
- Multiple sparsity levels handle different distance ranges

### 2. **Choose the Right Database**
- Graph databases (Neo4j) for graph traversals
- Native graph DB beats SQL for constant-time edge traversal
- In-memory caching for hot data (sparse graphs)

### 3. **Geohash Partitioning**
- Natural fit for location-based data
- Keeps related data together
- Handle hot spots with dynamic sizing or replicas

### 4. **Stream Processing for Real-time Updates**
- Kafka + Flink for stateful stream processing
- Partition alignment is crucial (geohash, road ID)
- Efficient data structures (linked lists) for rolling averages

### 5. **Dependency Management**
- Shortcut edges create complex dependencies
- Track dependencies in relational DB
- Use CDC to propagate changes
- Only recalculate when threshold exceeded

### 6. **Hidden Markov Models**
- GPS is imperfect; need probabilistic road matching
- Viterbi algorithm finds most likely path
- Requires stateful processing (Flink)

### 7. **Trade-offs Everywhere**
- Best route vs. fast response
- Storage cost vs. query speed
- Update frequency vs. system load
- Replication vs. write complexity

---

## Further Optimizations (Not Covered)

- **A* algorithm** instead of Dijkstra (uses heuristics)
- **Turn restrictions** and road rules
- **Multi-modal routing** (walking + transit)
- **Historical traffic patterns** for prediction
- **Machine learning** for better ETA predictions
- **Edge caching** for common routes
- **Pre-computed routes** for popular destinations

---

## Conclusion

Google Maps is a masterclass in:
- **Algorithm optimization** (contraction hierarchies)
- **Data structure selection** (graph databases, in-memory caches)
- **Stream processing** (real-time updates with Kafka/Flink)
- **Distributed systems** (partitioning, replication, sharding)
- **Probabilistic modeling** (Hidden Markov Models)

The system balances accuracy, latency, and scale through careful architectural choices and algorithmic innovations.

---

## Real-World Example: "Home → Office" (3 km Route)

Let's walk through how Google Maps (or any large-scale navigation system) handles your 3 km "Home → Office" route request, step by step.

We'll go from your phone → backend → algorithms → final ETA — everything simplified but accurate.

### Step-by-Step Process

#### 1️⃣ You Enter the Destination

You open Google Maps and type "Office".

The app **geocodes** your input:
- "Office" → (lat₁, long₁) for home and (lat₂, long₂) for office

**Example**:
```
Home: 12.9356, 77.6140
Office: 12.9621, 77.6378
```

---

#### 2️⃣ Route Request Sent to Backend

Your phone sends:

```json
{
  "source": [12.9356, 77.6140],
  "destination": [12.9621, 77.6378],
  "mode": "driving",
  "time": "now"
}
```

This request hits Google's **Route Service** (Load balanced → distributed system).

---

#### 3️⃣ Find Nearby Road Nodes

The backend maps both coordinates to the nearest graph nodes (intersections or road points).

Uses **Geohash partitioning**:
- Nearby locations (same 6–7 character geohash) → same shard

**Example**:
```
Source → Node 10123
Destination → Node 10487
```

Each node and road segment (edge) is stored in **Neo4j** (graph DB) — or an in-memory sparse graph if it's a short route.

---

#### 4️⃣ Graph Selection (Local Route = Level 1 Graph)

Since **3 km < 10 min**, Google Maps uses **Level 1: Full Graph** (all local roads).

For long trips (>2 hrs), it would switch to sparse or super-sparse graphs (highways only).

So, it loads all local intersections and roads within ~5–10 km radius.

---

#### 5️⃣ Run Dijkstra / A* Search

Now, it runs **A*** (optimized Dijkstra) between the two nodes.

Each edge has:
- **Base weight** = road length / speed limit
- **Dynamic weight** = updated using real-time speed (from live drivers' GPS)

**Example**:

| Road   | Distance | Speed   | Time (min) |
|--------|----------|---------|------------|
| A → B  | 1.0 km   | 30 km/h | 2.0        |
| B → C  | 1.5 km   | 25 km/h | 3.6        |
| C → D  | 0.5 km   | 20 km/h | 1.5        |

Algorithm picks the lowest cumulative time path:

```
Home (A) → B → C → D (Office)
Total = 7.1 minutes
```

---

#### 6️⃣ Live Traffic (Kafka + Flink)

While you're routing:

Phones continuously send **GPS pings**:
```json
{ "lat": 12.9356, "lon": 77.6140, "timestamp": "2025-11-03T10:30:00Z" }
```

A **Hidden Markov Model (HMM)** running on Flink maps each GPS point to the most probable road segment.

Average speeds are updated every **1–5 minutes** per road using:
- **Kafka** (message bus)
- **Flink** (stream aggregator)
- **Neo4j** (graph update)

If a road near you slows down, its edge weight increases.

→ **ETA updates live** while you drive.

---

#### 7️⃣ ETA Calculation

```
ETA = (Sum of all edge travel times) + (Traffic factor) + (Signal delays)
```

**Example**:
```
Base ETA: 7.1 min
Traffic adjustment: +1.2 min
Signals & local delay: +0.7 min
Final ETA ≈ 9 min
```

---

#### 8️⃣ Path Rendering

Once the backend computes the route:

It returns an ordered list of coordinates:

```json
[
  [12.9356, 77.6140],
  [12.9401, 77.6185],
  [12.9450, 77.6230],
  [12.9621, 77.6378]
]
```

The app:
- Renders it as a **polyline overlay** on the map
- The voice assistant begins **turn-by-turn navigation**

---

### Summary Diagram

```
📱 User → Route Request → 🌐 Route Service
         ↓
   Geo-mapping (nearest nodes)
         ↓
   Dijkstra/A* on Local Graph (Neo4j/In-memory)
         ↓
   Uses live edge weights (Kafka + Flink)
         ↓
   Compute Best Path + ETA
         ↓
   Return Polyline + Instructions
         ↓
📱 Display route, voice navigation
```

---

### Key Design Learnings Applied

| Concept | Description |
|---------|-------------|
| **Graph Hierarchy** | Uses full/sparse/super-sparse depending on distance |
| **Geohash Partitioning** | Keeps data for nearby roads on same shard |
| **Dijkstra / A*** | Finds shortest path based on travel time |
| **Kafka + Flink** | Stream GPS → real-time speed updates |
| **Neo4j** | Graph storage with O(1) edge traversal |
| **CDC (MySQL + Flink)** | Updates dependent shortcut edges when base edge speed changes |

---

### 🚀 Why It's Fast

Even though the full city graph may have millions of nodes:

- Only your **local area** (1 partition) is searched
- Edges have **precomputed shortcuts** (via contraction hierarchies)
- **In-memory caching** ensures sub-second response

**Result**: A route for 3 km is computed in **< 200 ms**

---
