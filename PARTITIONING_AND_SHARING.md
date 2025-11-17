# Partitioning and Sharing in System Design

## Overview

**Partitioning** (also called **Sharding**) is the practice of dividing data across multiple nodes/servers to achieve horizontal scalability. **Sharing** refers to how multiple clients or services access and utilize these partitioned resources.

This document explores 3 real-world use cases demonstrating different partitioning and sharing strategies.

---

## Use Case 1: Database Sharding for User Data

### Scenario
A social media platform with 100 million users needs to scale its user profile database. A single database server cannot handle the read/write load and storage requirements.

### Partitioning Strategy: Hash-based Sharding by User ID

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                   │
├─────────────────┬───────────────────┬───────────────────────────────────┤
│                 │                   │                                   │
│  ┌──────────┐   │  ┌──────────┐     │    ┌──────────┐                  │
│  │ Mobile   │   │  │   Web    │     │    │   API    │                  │
│  │   App    │   │  │   App    │     │    │ Service  │                  │
│  └─────┬────┘   │  └─────┬────┘     │    └─────┬────┘                  │
│        │        │        │          │          │                       │
└────────┼────────┴────────┼──────────┴──────────┼───────────────────────┘
         │                 │                     │
         │                 │                     │
         └─────────────────┼─────────────────────┘
                           │
                           ▼
         ┌─────────────────────────────────────────────┐
         │      APPLICATION LAYER - SHARD ROUTER       │
         │                                             │
         │   Hash Function: shard = user_id % 4        │
         │   Routes requests to correct database       │
         └──┬────────┬─────────┬────────┬─────────────┘
            │        │         │        │
  ┌─────────┘        │         │        └──────────┐
  │   user_id % 4=0  │         │  user_id % 4=3    │
  │                  │         │                   │
  ▼                  ▼         ▼                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       DATABASE SHARDS                               │
├──────────────┬──────────────┬──────────────┬──────────────────────┤
│   SHARD 0    │   SHARD 1    │   SHARD 2    │      SHARD 3         │
├──────────────┼──────────────┼──────────────┼──────────────────────┤
│  ╔════════╗  │  ╔════════╗  │  ╔════════╗  │    ╔════════╗        │
│  ║  DB 0  ║  │  ║  DB 1  ║  │  ║  DB 2  ║  │    ║  DB 3  ║        │
│  ║        ║  │  ║        ║  │  ║        ║  │    ║        ║        │
│  ║ Users: ║  │  ║ Users: ║  │  ║ Users: ║  │    ║ Users: ║        │
│  ║ 0,4,8  ║  │  ║ 1,5,9  ║  │  ║ 2,6,10 ║  │    ║ 3,7,11 ║        │
│  ║ 12,16  ║  │  ║ 13,17  ║  │  ║ 14,18  ║  │    ║ 15,19  ║        │
│  ║  ...   ║  │  ║  ...   ║  │  ║  ...   ║  │    ║  ...   ║        │
│  ║        ║  │  ║        ║  │  ║        ║  │    ║        ║        │
│  ║ 25M    ║  │  ║ 25M    ║  │  ║ 25M    ║  │    ║ 25M    ║        │
│  ║ users  ║  │  ║ users  ║  │  ║ users  ║  │    ║ users  ║        │
│  ╚════════╝  │  ╚════════╝  │  ╚════════╝  │    ╚════════╝        │
│              │              │              │                      │
│  Server 1    │  Server 2    │  Server 3    │    Server 4          │
└──────────────┴──────────────┴──────────────┴──────────────────────┘
```

### Data Distribution Example

```
User ID → Shard Mapping:

user_id: 12345  →  12345 % 4 = 1  →  Stored on SHARD 1
user_id: 99999  →  99999 % 4 = 3  →  Stored on SHARD 3
user_id: 88888  →  88888 % 4 = 0  →  Stored on SHARD 0
user_id: 77777  →  77777 % 4 = 1  →  Stored on SHARD 1

Result: Even distribution across all 4 shards
```

### Key Concepts

**Partitioning:**
- Data is divided using hash function: `shard = user_id % 4`
- Each shard holds ~25M users
- User data is stored on exactly one shard (no duplication)
- Even distribution of data across shards

**Sharing:**
- All clients share access to the same logical database
- Shard router directs requests to the correct physical shard
- Clients are unaware of the partitioning scheme
- Read/write operations distributed across all shards

### Advantages
✓ Horizontal scalability (add more shards as users grow)
✓ Parallel query execution across shards
✓ Isolation of failures (one shard down doesn't affect others)
✓ Even load distribution

### Challenges
✗ Cross-shard queries are expensive (e.g., "find all users named John")
✗ Rebalancing data when adding/removing shards
✗ Maintaining referential integrity across shards

### Example Query Flow

```
Step-by-Step: Get user profile for user_id = 12345

┌─────────┐
│ Client  │  1. Request: GET /user/12345
└────┬────┘
     │
     ▼
┌──────────────────┐
│  Shard Router    │  2. Calculate: 12345 % 4 = 1
└────┬─────────────┘     Route to Shard 1
     │
     ▼
┌──────────────────┐
│    Shard 1       │  3. Query: SELECT * FROM users WHERE id=12345
│   (Server 2)     │  4. Return: {id: 12345, name: "John", ...}
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│  Shard Router    │  5. Forward response
└────┬─────────────┘
     │
     ▼
┌─────────┐
│ Client  │  6. Receive user profile
└─────────┘
```

---

## Use Case 2: Distributed Cache with Consistent Hashing

### Scenario
An e-commerce platform needs to cache product information, user sessions, and shopping carts. The cache must handle 1 million requests/second with minimal latency and gracefully handle node failures.

### Partitioning Strategy: Consistent Hashing

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                                  │
├────────────┬────────────┬────────────┬──────────────────────────────┤
│            │            │            │                              │
│ ┌────────┐ │ ┌────────┐ │ ┌────────┐ │  ┌────────┐                 │
│ │  Web   │ │ │  Web   │ │ │  Web   │ │  │ Mobile │                 │
│ │Server 1│ │ │Server 2│ │ │Server 3│ │  │  API   │                 │
│ └───┬────┘ │ └───┬────┘ │ └───┬────┘ │  └───┬────┘                 │
│     │      │     │      │     │      │      │                      │
└─────┼──────┴─────┼──────┴─────┼──────┴──────┼──────────────────────┘
      │            │            │             │
      └────────────┼────────────┼─────────────┘
                   │            │
                   ▼            ▼
      ┌────────────────────────────────────────┐
      │   CACHE CLIENT LIBRARY (embedded)      │
      │                                        │
      │   • Maintains consistent hash ring     │
      │   • Tracks node health                 │
      │   • Routes keys to correct node        │
      └────┬──────┬──────┬──────┬─────────────┘
           │      │      │      │
           │      │      │      │
    ┌──────┘      │      │      └──────┐
    │             │      │             │
    ▼             ▼      ▼             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CACHE NODE CLUSTER                           │
├──────────────┬──────────────┬──────────────┬───────────────────┤
│  CACHE       │  CACHE       │  CACHE       │   CACHE           │
│  NODE 1      │  NODE 2      │  NODE 3      │   NODE 4          │
├──────────────┼──────────────┼──────────────┼───────────────────┤
│  ┌────────┐  │  ┌────────┐  │  ┌────────┐  │   ┌────────┐      │
│  │Products│  │  │Products│  │  │Products│  │   │Products│      │
│  │  A-G   │  │  │  H-M   │  │  │  N-S   │  │   │  T-Z   │      │
│  ├────────┤  │  ├────────┤  │  ├────────┤  │   ├────────┤      │
│  │Sessions│  │  │ Carts  │  │  │Sessions│  │   │ Carts  │      │
│  ├────────┤  │  ├────────┤  │  ├────────┤  │   ├────────┤      │
│  │ 10GB   │  │  │ 10GB   │  │  │ 10GB   │  │   │ 10GB   │      │
│  │  RAM   │  │  │  RAM   │  │  │  RAM   │  │   │  RAM   │      │
│  └────────┘  │  └────────┘  │  └────────┘  │   └────────┘      │
│              │              │              │                   │
│  Port: 6379  │  Port: 6380  │  Port: 6381  │   Port: 6382      │
└──────┬───────┴──────────────┴──────┬───────┴───────────────────┘
       │                             │
       │  Async Replication          │  Async Replication
       ▼                             ▼
  ┌──────────┐                  ┌──────────┐
  │ Replica  │                  │ Replica  │
  │  Node 1  │                  │  Node 2  │
  └──────────┘                  └──────────┘
```

### Visual: Consistent Hash Ring

```
                        0° / 360°
                      (Cache Node 1)
                           *
                          /|\
                         / | \
                        /  |  \
                       /   |   \
                      /    |    \
                     /     |     \
            315°    /      |      \    45°
        (cart:xyz) /       |       \ (product:12345)
                  /        |        \
                 /         |         \
                /          |          \
               /           |           \
              *            |            *
        270°               |               90°
                           |          (Cache Node 2)
                           |
                           |
                           *
                         180°
                    (Cache Node 3)
                   (session:abc)


Key Distribution:
┌────────────────┬──────────────┬──────────────────────────┐
│ Key            │ Hash Result  │ Stored On                │
├────────────────┼──────────────┼──────────────────────────┤
│ product:12345  │    45°       │ Node 2 (first clockwise) │
│ session:abc    │   200°       │ Node 3 (first clockwise) │
│ cart:xyz       │   315°       │ Node 1 (first clockwise) │
│ product:99999  │   120°       │ Node 3 (first clockwise) │
└────────────────┴──────────────┴──────────────────────────┘

When Node 2 FAILS:
    Before: Keys from 0°-90° → Node 1, 90°-180° → Node 2
    After:  Keys from 0°-180° → ALL go to Node 3
    Impact: Only 25% of total keys need remapping!
```

### Key Concepts

**Partitioning:**
- Each cache key is hashed to a position on a virtual ring (0-360°)
- Cache nodes are placed at specific positions on the ring
- A key is stored on the first node encountered clockwise from its position
- Virtual nodes (multiple positions per physical node) ensure better distribution

**Sharing:**
- Multiple application servers share the same cache cluster
- Client library maintains the hash ring topology
- Automatic failover: if a node fails, its keys move to the next node
- Write-through or write-behind patterns for data consistency

### Advantages
✓ Minimal data movement when nodes are added/removed (only 1/N keys affected)
✓ Fault tolerant with automatic failover
✓ Distributed load across all nodes
✓ Client-side routing (no single point of failure)

### Challenges
✗ Cache coherence across replicas
✗ Thundering herd problem when cache expires
✗ Memory pressure on nodes after failures

### Example: Handling Node Failure

```
TIMELINE: Cache Node 2 Failure

t=0: Normal Operation (4 nodes)
     ┌──────┬──────┬──────┬──────┐
     │Node 1│Node 2│Node 3│Node 4│
     │ 25%  │ 25%  │ 25%  │ 25%  │  ← Each handles 25% of keys
     └──────┴──────┴──────┴──────┘

t=1: Node 2 Crashes
     ┌──────┬──────┬──────┬──────┐
     │Node 1│ XXXX │Node 3│Node 4│
     │ 25%  │ FAIL │ 25%  │ 25%  │
     └──────┴──────┴──────┴──────┘

t=2: Client Library Detects Failure (timeout/health check)
     • Removes Node 2 from hash ring
     • Recalculates key positions

t=3: Keys Redistributed
     ┌──────┬──────┬──────┐
     │Node 1│Node 3│Node 4│
     │ 25%  │ 50%  │ 25%  │  ← Node 3 now handles Node 2's traffic
     └──────┴──────┴──────┘

t=4: Cache Repopulation
     • Cache misses for old Node 2 keys
     • Data fetched from database
     • Node 3 gradually fills with data

Result: 75% of requests unaffected, system continues operating!
```

---

## Use Case 3: Message Queue Partitioning (Kafka-style)

### Scenario
A ride-sharing app needs to process real-time events: ride requests, driver location updates, payment transactions. The system must handle 100K events/second while maintaining event ordering per user/driver and enabling parallel processing.

### Partitioning Strategy: Topic Partitions with Consumer Groups

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         EVENT PRODUCERS                                  │
├────────────────────┬────────────────────┬────────────────────────────────┤
│                    │                    │                                │
│  ┌──────────────┐  │  ┌──────────────┐  │    ┌──────────────┐           │
│  │  Mobile App  │  │  │  Driver App  │  │    │   Payment    │           │
│  │              │  │  │              │  │    │   Service    │           │
│  │ Publishes:   │  │  │ Publishes:   │  │    │              │           │
│  │ • ride req   │  │  │ • location   │  │    │ Publishes:   │           │
│  │              │  │  │ • status     │  │    │ • txn events │           │
│  └───────┬──────┘  │  └───────┬──────┘  │    └───────┬──────┘           │
│          │         │          │         │            │                  │
└──────────┼─────────┴──────────┼─────────┴────────────┼──────────────────┘
           │                    │                      │
           │ (partition key:    │  (partition key:     │ (partition key:
           │  user_id)          │   driver_id)         │  user_id)
           │                    │                      │
           ▼                    ▼                      ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    MESSAGE QUEUE CLUSTER                                 │
│                    Topic: "ride-events"                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────┬────────────────────┬───────────────────────┐    │
│  │   PARTITION 0      │   PARTITION 1      │   PARTITION 2         │    │
│  │ user_id % 4 = 0    │ user_id % 4 = 1    │ user_id % 4 = 2       │    │
│  │ Leader: Broker 1   │ Leader: Broker 2   │ Leader: Broker 3      │    │
│  ├────────────────────┼────────────────────┼───────────────────────┤    │
│  │ [E1][E2][E3][E4]   │ [E1][E2][E3][E4]   │ [E1][E2][E3][E4]      │    │
│  │  ↑                 │  ↑                 │  ↑                    │    │
│  │ Offset: 0→         │ Offset: 0→         │ Offset: 0→            │    │
│  │                    │                    │                       │    │
│  │ Users: 0,4,8,12    │ Users: 1,5,9,13    │ Users: 2,6,10,14      │    │
│  └─────────┬──────────┴──────────┬─────────┴──────────┬────────────┘    │
│            │                     │                    │                 │
│  ┌─────────┴───────────────────────────────────────────┴─────────┐      │
│  │                      PARTITION 3                              │      │
│  │                   user_id % 4 = 3                             │      │
│  │                   Leader: Broker 1                            │      │
│  ├───────────────────────────────────────────────────────────────┤      │
│  │                   [E1][E2][E3][E4]                            │      │
│  │                    ↑                                          │      │
│  │                   Offset: 0→                                  │      │
│  │                                                               │      │
│  │                   Users: 3,7,11,15                            │      │
│  └───────────────────────────────────────────────────────────────┘      │
│                                                                          │
└──┬────────────────┬───────────────┬──────────────┬────────────────────┬─┘
   │                │               │              │                    │
   │                │               │              │                    │
   ▼                ▼               │              │                    │
┌─────────────────────────────────┐ │              │                    │
│  CONSUMER GROUP: ride-processors│ │              │                    │
├─────────────────────────────────┤ │              │                    │
│                                 │ │              │                    │
│  ┌───────────┐  ┌────────────┐  │ │              │                    │
│  │Consumer 1 │  │Consumer 2  │  │ │              │                    │
│  │           │  │            │  │ │              │                    │
│  │Reads:     │  │Reads:      │  │ │              │                    │
│  │Part 0     │  │Part 1      │  │ │              │                    │
│  └───────────┘  └────────────┘  │ │              │                    │
│                                 │ │              │                    │
│  ┌───────────────────────────┐  │ │              │                    │
│  │      Consumer 3           │  │ │              │                    │
│  │                           │  │ │              │                    │
│  │ Reads: Part 2, Part 3     │  │ │              │                    │
│  └───────────────────────────┘  │ │              │                    │
└─────────────────────────────────┘ │              │                    │
                                    ▼              ▼                    ▼
                              ┌──────────────────────────────────────────┐
                              │ CONSUMER GROUP: analytics                │
                              ├──────────────────────────────────────────┤
                              │                                          │
                              │  ┌───────────────────────────────────┐   │
                              │  │      Analytics Consumer 1         │   │
                              │  │      Reads: Part 0, Part 1        │   │
                              │  └───────────────────────────────────┘   │
                              │                                          │
                              │  ┌───────────────────────────────────┐   │
                              │  │      Analytics Consumer 2         │   │
                              │  │      Reads: Part 2, Part 3        │   │
                              │  └───────────────────────────────────┘   │
                              │                                          │
                              │  (Independent offset tracking)           │
                              └──────────────────────────────────────────┘
```

### Partition Data Structure

```
Topic: ride-events (4 partitions)

PARTITION 0 (user_id % 4 = 0):
┌──────────────────────────────────────────────────────────────┐
│ Offset │ Timestamp           │ Key (user_id) │ Event        │
├────────┼─────────────────────┼───────────────┼──────────────┤
│   0    │ 2025-11-17 10:00:00 │ user_4        │ ride_request │
│   1    │ 2025-11-17 10:00:05 │ user_4        │ ride_confirm │
│   2    │ 2025-11-17 10:01:00 │ user_8        │ ride_request │
│   3    │ 2025-11-17 10:02:00 │ user_12       │ ride_request │
│  ...   │        ...          │      ...      │     ...      │
└────────┴─────────────────────┴───────────────┴──────────────┘
          ↑
    Consumer 1 reads here (maintains own offset)


PARTITION 1 (user_id % 4 = 1):
┌──────────────────────────────────────────────────────────────┐
│ Offset │ Timestamp           │ Key (user_id) │ Event        │
├────────┼─────────────────────┼───────────────┼──────────────┤
│   0    │ 2025-11-17 10:00:01 │ user_1        │ ride_request │
│   1    │ 2025-11-17 10:00:10 │ user_5        │ ride_request │
│   2    │ 2025-11-17 10:01:05 │ user_1        │ ride_complete│
│   3    │ 2025-11-17 10:02:30 │ user_9        │ ride_request │
│  ...   │        ...          │      ...      │     ...      │
└────────┴─────────────────────┴───────────────┴──────────────┘
          ↑
    Consumer 2 reads here

ORDERING GUARANTEE: All events for user_1 are in order within Partition 1!
```

### Key Concepts

**Partitioning:**
- Topic is divided into multiple ordered partitions
- Messages with same partition key go to same partition
- Each partition is an ordered, immutable log
- Partitions can be replicated across brokers for fault tolerance

**Sharing:**
- Multiple consumer groups can read the same topic independently
- Within a consumer group, each partition is assigned to exactly one consumer
- Enables both parallel processing and multiple independent workloads
- Each consumer group maintains its own offset (read position)

### Advantages
✓ Parallel processing (scale by adding consumers)
✓ Ordering guarantee per partition key (e.g., all events for user_123 are ordered)
✓ Multiple consumer groups enable different use cases (processing + analytics)
✓ Replay capability (consumers can rewind to any offset)
✓ High throughput (distributed writes and reads)

### Challenges
✗ Number of partitions is typically fixed at topic creation
✗ Hot partitions if keys are not evenly distributed
✗ Consumer rebalancing when adding/removing consumers
✗ No ordering across partitions

### Example: Event Processing Flow

```
Event: User 12345 requests a ride

STEP 1: Producer Publishes Event
┌──────────────────────────────────────┐
│ {                                    │
│   "user_id": 12345,                  │
│   "event": "ride_requested",         │
│   "location": {lat: 37.7, lon: -122},│
│   "timestamp": "2025-11-17T10:00:00" │
│ }                                    │
└──────────────────────────────────────┘
        │
        │ Partition key: user_id = 12345
        ▼
┌──────────────────────────────────────┐
│ Partition Calculation:               │
│ partition = hash(12345) % 4 = 1      │
└──────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│ PARTITION 1 (on Broker 2)            │
│                                      │
│ Append to log:                       │
│ • Assigned offset: 45023             │
│ • Write to disk                      │
│ • Replicate to 2 replica brokers     │
└──────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│ TWO CONSUMER GROUPS READ SAME EVENT (independent offsets) │
└────────────────────────────────────────────────────────────┘
        │
        ├──────────────────────────────┬─────────────────────┐
        ▼                              ▼                     ▼
┌──────────────────┐        ┌──────────────────┐   ┌──────────────────┐
│  Consumer 2      │        │  Analytics 1     │   │  Audit Logger    │
│  (ride-processor)│        │  (analytics grp) │   │  (audit group)   │
├──────────────────┤        ├──────────────────┤   ├──────────────────┤
│ 1. Fetch from    │        │ 1. Fetch same    │   │ 1. Fetch same    │
│    offset 45020  │        │    event         │   │    event         │
│                  │        │                  │   │                  │
│ 2. Process event │        │ 2. Update real-  │   │ 2. Store to      │
│    • Match driver│        │    time dashboard│   │    audit log     │
│    • Send notif  │        │                  │   │                  │
│                  │        │ 3. Commit offset │   │ 3. Commit offset │
│ 3. Commit offset │        │    45023 (indep) │   │    45023 (indep) │
│    45023         │        │                  │   │                  │
└──────────────────┘        └──────────────────┘   └──────────────────┘

Each consumer group maintains INDEPENDENT progress!
```

### Consumer Rebalancing Scenario

```
SCENARIO: Scaling Consumer Group

INITIAL STATE (3 consumers, 4 partitions):
┌──────────────────────────────────────────────────────────────┐
│  Consumer Group: ride-processors                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Consumer 1  ───────────> Partition 0  (25% of events)      │
│                                                              │
│  Consumer 2  ───────────> Partition 1  (25% of events)      │
│                                                              │
│  Consumer 3  ───┬──────> Partition 2  (25% of events)       │
│                 └──────> Partition 3  (25% of events)       │
│                                                              │
│  Issue: Consumer 3 handling 50% of load!                    │
└──────────────────────────────────────────────────────────────┘


EVENT: New Consumer 4 Joins
┌──────────────────────────────────────────────────────────────┐
│ 1. Consumer 4 sends JoinGroup request                        │
│ 2. Coordinator triggers REBALANCE                            │
│ 3. All consumers STOP processing                             │
│ 4. Coordinator reassigns partitions                          │
└──────────────────────────────────────────────────────────────┘


NEW STATE (4 consumers, 4 partitions):
┌──────────────────────────────────────────────────────────────┐
│  Consumer Group: ride-processors (after rebalance)           │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Consumer 1  ───────────> Partition 0  (25% of events)      │
│                                                              │
│  Consumer 2  ───────────> Partition 1  (25% of events)      │
│                                                              │
│  Consumer 3  ───────────> Partition 2  (25% of events)      │
│                                                              │
│  Consumer 4  ───────────> Partition 3  (25% of events)  ✓   │
│                                                              │
│  Result: Perfectly balanced load distribution!               │
└──────────────────────────────────────────────────────────────┘

TIMELINE:
t=0:    Consumer 4 joins
t=1-2:  Rebalancing (brief pause in processing)
t=3:    All consumers resume from last committed offset
t=4+:   25% better throughput, lower latency per consumer
```

---

## Comparison Matrix

| Aspect | Database Sharding | Distributed Cache | Message Queue |
|--------|------------------|-------------------|---------------|
| **Partitioning Method** | Hash-based | Consistent Hashing | Topic Partitions |
| **Primary Goal** | Scale storage & writes | Low latency reads | Event streaming & ordering |
| **Data Distribution** | Even distribution | Minimal redistribution on changes | Key-based ordering |
| **Consistency Model** | Strong (per shard) | Eventual | Eventual (ordered per partition) |
| **Sharing Pattern** | Shared-nothing | Shared cache with local clients | Shared topic, partitioned consumption |
| **Typical Use Case** | User profiles, transactional data | Sessions, product catalog | Events, logs, real-time data |
| **Failure Handling** | Failover to replica | Consistent hashing redistribution | Consumer rebalancing |
| **Scalability** | Add shards (complex) | Add nodes (simple) | Add consumers (automatic) |
| **Ordering Guarantee** | No | No | Yes (per partition) |

---

## Best Practices

### Choosing a Partitioning Strategy

1. **Hash Partitioning**: Use when you need even distribution and mostly single-record access
   - Example: User profiles, product catalog
   - Pro: Even distribution, simple logic
   - Con: Hard to rebalance

2. **Range Partitioning**: Use when you frequently query ranges of data
   - Example: Time-series data, logs by date
   - Pro: Range queries efficient
   - Con: Hot spots on recent data

3. **Consistent Hashing**: Use when nodes frequently join/leave the cluster
   - Example: Distributed caches, CDN
   - Pro: Minimal data movement
   - Con: Complex implementation

4. **Key-based Partitioning**: Use when ordering matters within a key
   - Example: Event streams, activity feeds
   - Pro: Ordering guaranteed
   - Con: Uneven distribution if keys skewed

### Sharing Considerations

- **Access Patterns**: Design partitions based on how data is accessed
- **Hotspots**: Monitor for uneven load; use composite keys if needed
- **Cross-partition Operations**: Minimize joins/aggregations across partitions
- **Monitoring**: Track partition sizes, request distribution, and rebalancing events

### Partition Key Selection

```
Good Partition Keys:
✓ user_id        - High cardinality, even distribution
✓ order_id       - Unique, randomly distributed
✓ device_id      - Many unique values

Bad Partition Keys:
✗ country        - Low cardinality (few values)
✗ date           - Creates hot spots on current date
✗ status         - Very low cardinality (active/inactive)

Composite Keys (when needed):
• country + user_id    - Balances locality with distribution
• date + order_id      - Enables time queries with good distribution
```

---

## Conclusion

Partitioning and sharing are fundamental to building scalable distributed systems. Each use case demonstrates different trade-offs:

- **Database Sharding**: Optimizes for transactional consistency and storage scalability
  - Best for: User data, profiles, transactional records
  - Trade-off: Complex cross-shard queries

- **Distributed Cache**: Optimizes for read latency and graceful degradation
  - Best for: Session data, frequently accessed content
  - Trade-off: Eventual consistency

- **Message Queue**: Optimizes for throughput, ordering, and parallel processing
  - Best for: Event streams, real-time data processing
  - Trade-off: Fixed partition count, rebalancing overhead

**Choose the strategy that aligns with your system's primary bottleneck and access patterns.**

### Quick Decision Guide

```
Start Here
    │
    ▼
Need to store data permanently? ─── YES ──> Database Sharding
    │
    NO
    │
    ▼
Need ordering guarantees? ─── YES ──> Message Queue Partitioning
    │
    NO
    │
    ▼
Need fast reads/temporary storage? ─── YES ──> Distributed Cache
```
