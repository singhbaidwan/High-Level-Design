# Real-Time Delivery Agent Matching Architecture (Redis Deep Dive)

## Overview

Matching a user or restaurant with a nearby delivery agent is **not a traditional search problem**.
It is a **real-time distributed coordination problem** that must satisfy:

* **Low latency (<100ms decision time)**
* **High throughput (millions of location updates)**
* **Strong correctness guarantees (no double assignment)**
* **Constant system availability**

Large delivery platforms (Uber Eats, DoorDash, Swiggy, Zomato, Instacart) implement a **real-time in-memory matching architecture** typically powered by **Redis**, because:

* It supports **in-memory operations**
* It provides **atomic primitives**
* It handles **high-frequency writes**
* It provides **native geospatial indexing**

This document explains the **complete architecture** from a **Staff Engineer perspective**, including:

* High-level architecture
* Data modeling
* Redis structures
* Matching algorithm
* Locking strategy
* Failure handling
* Scalability strategies
* Tradeoffs

---

# 1. Problem Definition

### Inputs

```
Pickup location
Thousands of moving delivery agents
Agent state:
    - available
    - assigned
    - offline
```

### Output

```
Exactly ONE agent assigned
Lowest ETA
<100ms decision latency
```

### Constraints

| Constraint           | Value             |
| -------------------- | ----------------- |
| Agents in city       | 50k – 200k        |
| Location update rate | every 1–3 seconds |
| Order creation       | thousands/sec     |
| Matching latency     | <100ms            |

---

# 2. Why This Is NOT a Search Problem

A search system (Elasticsearch, Solr) is optimized for:

```
Query → Static Index → Results
```

But delivery matching involves **rapidly changing state**.

### Key Characteristics

| Property           | Value               |
| ------------------ | ------------------- |
| Agent location     | constantly changing |
| Agent availability | constantly changing |
| Assignments        | must be atomic      |
| Decisions          | real-time           |

Search engines fail because:

* Index updates are **slow**
* Writes cause **reindexing**
* No **atomic assignment guarantees**
* Latency > 100ms

Thus we use **Redis in-memory coordination**.

---

# 3. High-Level Architecture

```
Driver App
   ↓
Location Updates
   ↓
Kafka / Streaming Pipeline
   ↓
Location Processor
   ↓
Redis Geo Index
```

Order flow:

```
User places order
      ↓
Order Service
      ↓
Matching Service
      ↓
Redis Query
      ↓
Assignment Lock
      ↓
Notification Service
      ↓
Driver App
```

### Key Architectural Principle

**Separate concerns**

```
Location
Availability
Assignment
```

These must **NOT be stored together**.

---

# 4. Redis Data Model

The system stores **three independent structures**.

---

# 4.1 Location Index (Geo)

Stores **driver coordinates only**.

```
GEOADD drivers:geo:blr 77.5946 12.9716 driver_123
```

Purpose:

```
Answer:
Who is geographically nearby?
```

Query example:

```
GEORADIUS drivers:geo:blr 77.5946 12.9716 3 km
```

Returns:

```
driver_123
driver_945
driver_554
```

Important:

This index **does NOT store availability**.

---

# 4.2 Availability Set

Stores which drivers **can receive orders**.

```
SADD drivers:available:blr driver_123
```

Remove when assigned:

```
SREM drivers:available:blr driver_123
```

Purpose:

```
Answer:
Who is eligible to receive an order?
```

---

# 4.3 Assignment Lock

Guarantees **no double assignment**.

```
SET driver:123:lock order_456 NX EX 10
```

Meaning:

```
Acquire lock only if not exists
Expire after 10 seconds
```

Purpose:

```
Answer:
Who wins the assignment?
```

---

# 5. Why Availability Must NOT Be Embedded in Geo Index

Bad design:

```
drivers:geo:available
drivers:geo:busy
drivers:geo:offline
```

This causes:

### Write Amplification

When a driver changes state:

```
Remove from one geo index
Insert into another
```

Location updates already occur every **1–3 seconds**.

Combining both leads to **exploding write volume**.

---

### Index Rebuild Problem

Geo indexes become unstable because:

```
availability change → geo rewrite
```

This kills Redis performance.

---

### Correct Design

Separate structures:

```
Location → GEO
Availability → SET
Assignment → LOCK
```

---

# 6. Spatial Segmentation (Geo Cells)

Cities can contain **100k+ drivers**.

Querying a single key:

```
drivers:geo:city
```

creates **hot keys**.

Instead we divide the map using **Geohash cells**.

Example:

```
tdr9k
tdr9m
tdr9n
```

Key structure:

```
drivers:geo:cell:tdr9k
drivers:available:cell:tdr9k
```

Benefits:

* smaller scans
* horizontal scaling
* Redis cluster friendly

---

# 7. Matching Flow (Detailed)

### Step 1 — Determine Pickup Cell

```
geo_cell = geohash(lat, lon)
```

---

### Step 2 — Expand Search Radius

Search strategy:

```
1km
2km
4km
```

Increasing radius prevents failure when driver density is low.

---

### Step 3 — Query Nearby Drivers

```
GEORADIUS drivers:geo:cell:tdr9k
```

Returns candidate drivers.

---

### Step 4 — Filter by Availability

```
available = SMEMBERS drivers:available:cell:tdr9k
```

Intersection:

```
candidates = nearby ∩ available
```

---

### Step 5 — Compute Ranking

Candidates ranked by:

```
ETA
Distance
Driver rating
Acceptance probability
```

ETA computed using:

```
road network + traffic model
```

---

### Step 6 — Attempt Assignment Lock

```
SET driver:123:lock order_456 NX EX 10
```

If success:

```
driver reserved
```

If failure:

```
driver already assigned
```

---

# 8. Sequential Assignment Strategy

Most platforms use **sequential dispatch** rather than broadcast.

### Why?

Broadcast causes:

```
Driver spam
Low acceptance
Chaos
```

Sequential dispatch:

```
Driver 1
Driver 2
Driver 3
```

Example:

```python
for driver in candidates:

    if acquire_lock(driver):

        send_order_request(driver)

        wait 5 seconds

        if accepted:
            assign(driver)
            break
        else:
            release_lock(driver)
```

Benefits:

```
Higher acceptance
Better ETA reliability
Lower system load
```

---

# 9. Handling Timeouts & Rejections

Drivers may:

```
Reject
Ignore
Lose network
```

Mechanism:

```
Lock TTL
```

Example:

```
SET driver:123:lock order_456 NX EX 10
```

If driver does nothing:

```
Lock auto expires
Driver returns to pool
```

---

# 10. Write Amplification Control

Drivers update location frequently.

Typical rate:

```
1–3 seconds
```

With 100k drivers:

```
~50k updates/sec
```

Optimizations:

### Movement Threshold

Update only if moved > 20 meters.

---

### Adaptive Frequency

```
Moving fast → update every 2s
Idle → update every 10s
```

---

### Batching

Mobile app sends batched updates.

---

# 11. Failure Handling

### Redis Failure

Solution:

```
Redis Cluster
Replica nodes
Automatic failover
```

---

### Matching Service Failure

Stateless services.

```
Kubernetes autoscaling
```

---

### Driver Disconnect

Heartbeat monitoring.

```
Last update > threshold
→ mark offline
```

---

# 12. Why Elasticsearch Is the Wrong Tool

| Capability            | Redis | Elasticsearch |
| --------------------- | ----- | ------------- |
| Latency               | <1ms  | 50–200ms      |
| Atomic operations     | Yes   | No            |
| High-frequency writes | Yes   | Poor          |
| Locks                 | Yes   | No            |
| TTL                   | Yes   | No            |

Elasticsearch is **excellent for search**, but **not real-time coordination**.

---

# 13. Scalability

At Uber scale:

```
Millions of drivers
Millions of orders
```

Scaling strategy:

### Redis Cluster

Partition by:

```
city
geo cell
```

---

### Matching Service Sharding

Each instance responsible for:

```
subset of geo cells
```

---

### Kafka Event Pipeline

Driver updates streamed through Kafka.

---

# 14. Advanced Optimizations

### Predictive Positioning

ML predicts where demand will appear.

Drivers repositioned proactively.

---

### Acceptance Probability Models

Rank drivers by:

```
acceptance likelihood
```

---

### Dynamic Radius Expansion

If no driver found:

```
increase radius
```

---

# 15. Observability

Metrics to monitor:

```
match_latency
driver_acceptance_rate
search_radius_growth
lock_conflicts
```

---

# 16. Key Architectural Principles

### 1. Separate Concerns

```
Location
Availability
Assignment
```

---

### 2. Favor Atomic Operations

Use Redis primitives:

```
SET NX
TTL
```

---

### 3. Minimize Write Amplification

Avoid rewriting geo indexes.

---

### 4. Prefer Sequential Assignment

Better driver experience.

---

# 17. Final Architecture Summary

```
Driver App
   ↓
Location Updates
   ↓
Kafka
   ↓
Location Processor
   ↓
Redis Geo Index
        ↓
Order Service
        ↓
Matching Service
        ↓
Availability Filter
        ↓
Assignment Lock
        ↓
Notification Service
        ↓
Driver App
```

---

# Closing Thoughts

Real-time delivery matching systems succeed because they:

```
Separate location, availability, assignment
Avoid expensive reindexing
Favor atomic operations
Operate fully in memory
```

The biggest lesson:

> Great architectures **compose simple data structures** rather than overloading one system.

This is why **Redis becomes the coordination backbone** of large real-time logistics platforms.

---
