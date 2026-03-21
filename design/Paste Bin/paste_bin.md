# PasteBin System Design (Staff-Level Interview Answer)

---

## 🧠 1. Problem Framing

PasteBin is **not just a URL shortener**. It is a hybrid system:

* Metadata service (short URL mapping)
* Large object storage system (paste content)
* CDN-backed read-heavy system

---

## 📌 Functional Requirements

* Create a paste → return short URL
* Fetch paste via short URL

---

## 📊 Scale & Constraints

* 600M writes/month
* 10B reads/month
* Avg size: 100KB
* Max size: 5GB

👉 This is a **read-heavy system (~17:1)** with **large immutable blobs**.

---

## 🏗️ 2. High-Level Architecture

### Components

* Load Balancers
* Stateless Application Servers
* Redis Cache (lookaside)
* URL Mapping Database
* Object Store (S3)
* CDN (CloudFront)

---

## 🔥 Key Design Principle

❌ Do NOT store paste content in DB
✅ Store in Object Store (S3)

### Why?

* Large rows → poor cache efficiency
* Long writes → lock contention
* Expensive replication
* High cross-AZ bandwidth cost

👉 Separation of concerns:

* **Control plane → DB (metadata)**
* **Data plane → Object Store (blobs)**

---

## 🔁 3. Write Flow (Create Paste)

### ❌ Naive Approach (Rejected)

```
CreatePaste(text, expiry)
```

Problem: pushes large payload through app servers

---

### ✅ Correct Approach

#### Step-by-step:

1. Client requests **pre-signed URL**
2. Server generates signed URL
3. Client uploads paste directly to S3
4. Client calls backend to create short URL

---

### 🎯 Why this works

* Removes server from data path
* Prevents network bottlenecks
* Improves scalability

---

### ⚠️ Order of Operations (Critical)

Correct:

1. Upload to S3
2. Then create short URL

---

### ❗ Why?

If reversed:

* DB may contain URL → but S3 upload fails
* Leads to broken links

---

### Trade-off

* May create **orphan S3 objects**

👉 Decision:

> Prefer **data correctness** over storage efficiency

---

## 📖 4. Read Flow (Fetch Paste)

### Flow:

1. Client sends short URL
2. App server fetches S3 URL (Redis → DB fallback)
3. Client fetches content via CDN
4. CDN → S3 (on cache miss)

---

## ⚡ Multi-Layer Caching Strategy

| Layer | Purpose                      |
| ----- | ---------------------------- |
| Redis | Fast metadata lookup         |
| CDN   | Cache large content globally |

---

### CDN Behavior

* Geo-distributed caching
* LRU eviction
* TTL ≈ 1 day
* Cache miss → fetch from S3

---

## 🧠 5. Data Model

```
shortURL → S3 URL, expiry
```

### Enhancements:

* Primary index on shortURL
* Expiry column
* Sharding using hash(shortURL)

---

## 🧨 6. Deep Dive: Cleaning Orphan S3 Files

### Problem

* Upload may succeed
* Short URL creation may fail

👉 Leaves unused objects in S3

---

### Cost Impact

* 600M pastes/month × 100KB = ~57TB/month
* Over time → millions in storage cost

---

### ❌ Bad Approach

* Query DB for expired entries
* Delete corresponding S3 objects

Problem:

* DB rows may be overwritten
* Lose reference to S3 objects

---

### ✅ Smart Solution (Key Insight)

Use **S3 object naming strategy**:

```
<timestamp>_<random>.txt
```

---

### How it works

* S3 lists objects lexicographically
* Timestamp prefix ensures sorted order
* Background job scans:

  * Stops when it hits non-expired items

---

### Benefits

* No DB dependency
* Efficient sequential scan
* Deterministic cleanup

---

## 🧠 7. API Design (Final)

```
CreateSignedURL(expiry) → s3URL
CreatePaste(s3URL, expiry) → shortURL
FetchPaste(shortURL) → s3URL
```

---

## 🔥 8. Staff-Level Trade-offs & Insights

### 1. Data Path Optimization

> Remove large payloads from app servers

---

### 2. Separation of Concerns

* DB → metadata
* S3 → content

---

### 3. Consistency Model

* Eventual consistency acceptable
* Immutable data simplifies system

---

### 4. Cost Awareness

* Storage cost dominates at scale
* Requires lifecycle or cleanup jobs

---

### 5. Failure Handling

* Partial failures handled via ordering
* Retry-safe design

---

## 🚀 9. Scaling Strategy

### Horizontal Scaling

* Stateless app servers
* Multiple load balancers

### Database Scaling

* Sharding by short URL
* Read replicas

### Caching

* Redis for hot metadata
* CDN for content

---

## 🎯 Final Staff-Level Summary

> “PasteBin is a read-heavy, blob-storage system where we decouple metadata from content, use direct-to-object-store uploads to eliminate server bottlenecks, and rely on CDN + caching for scale. We prioritize correctness over storage efficiency and handle cleanup asynchronously using deterministic object naming.”

---

## ✅ What differentiates a Staff answer

* Explicit trade-offs
* Cost considerations
* Failure handling
* Clear separation of data/control planes
* Deep understanding of infra constraints

---

💡 This is the level expected for:

* Google L5/L6
* Amazon SDE II / Senior
* Staff Engineer roles

---
