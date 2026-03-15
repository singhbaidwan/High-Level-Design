# Distributed Logging Framework Design

### System Design Interview (Staff Engineer Level)

Design a **logging framework used by many services producing millions of logs per second**.

The system should reliably ingest, process, store and query logs containing:

* timestamp
* service name
* log message
* metadata

---

# 1. Requirements (~5 minutes)

## Functional Requirements

Clients (microservices) should be able to:

1. **Send logs** to the logging system.
2. Each log must include:

   * timestamp
   * service name
   * log level
   * message
3. Logs should be **searchable later**.
4. Logs should support **filtering by service, time range, and log level**.
5. Logs should support **real-time monitoring dashboards**.

Optional features (if time allows):

* structured logging (JSON)
* distributed tracing correlation
* alerting based on logs

---

## Non Functional Requirements

### 1. High Throughput

Millions of logs per second.

Example:

```
10k services
100 logs/sec/service
= 1M logs/sec
```

System must handle **1M–5M writes/sec**.

---

### 2. High Availability

Logs are critical for:

* debugging production
* security audit
* incident analysis

Target:

```
99.99% availability
```

---

### 3. Durability

Logs should **not be lost during failures**.

Strategy:

* replication
* persistent queues
* multi-AZ storage

---

### 4. Low Ingestion Latency

Services must not block while logging.

Target:

```
< 10 ms logging overhead
```

---

### 5. Horizontal Scalability

System must scale with:

* number of services
* log volume

Architecture must support:

```
horizontal scaling
```

---

# 2. Capacity Estimation (Quick)

Assume:

```
1M logs/sec
Average log size = 500 bytes
```

### Write throughput

```
1M * 500B = 500MB/sec
```

Per day:

```
500MB * 86400
= 43 TB/day
```

Storage retention example:

```
30 days → 1.3 PB
```

Implication:

We **cannot store logs in a traditional relational DB**.

We need:

* distributed storage
* log compression
* cold storage

---

# 3. Core Entities (~2 minutes)

### LogEntry

```
LogEntry
---------
timestamp
service_name
log_level
message
trace_id
metadata
```

---

### Service

```
Service
---------
service_name
team
environment
version
```

---

### LogBatch

Used for efficient ingestion.

```
LogBatch
---------
batch_id
service_name
logs[]
timestamp
```

---

# 4. System Interface / APIs (~5 minutes)

Services interact using **logging SDKs**.

Example languages:

```
Java
Python
Go
NodeJS
```

---

### Log Ingestion API

```
POST /logs
```

Request:

```json
{
  "service": "payment-service",
  "logs": [
    {
      "timestamp": 171000000,
      "level": "ERROR",
      "message": "Payment failed",
      "trace_id": "abc123"
    }
  ]
}
```

---

### Query Logs

```
GET /logs?service=payment&start=...&end=...
```

---

### Stream Logs (Real-time)

```
GET /logs/stream
```

Uses:

```
WebSocket
```

---

# 5. Data Flow (~5 minutes)

### Log Write Flow

```
Service
   |
Logging SDK
   |
Log Collector
   |
Kafka
   |
Log Processing Service
   |
Log Storage
```

---

### Log Query Flow

```
User Dashboard
     |
Query API
     |
Log Index (Elasticsearch)
     |
Log Storage
```

---

# 6. High Level Architecture (~10–15 minutes)

## Component Overview

```
Services
   |
Logging SDK
   |
Log Gateway / Collector
   |
Message Queue (Kafka)
   |
Stream Processing Layer
   |
Storage Layer
   |
Search Engine
   |
Dashboard / Alerting
```

---

## 1. Logging SDK

Each service integrates SDK.

Responsibilities:

* attach timestamp
* attach service name
* batching
* retries
* compression

Example log:

```json
{
 "timestamp":171000,
 "service":"checkout-service",
 "level":"ERROR",
 "message":"Payment timeout"
}
```

---

### Why SDK?

Benefits:

* standard format
* reduces network overhead
* handles retries

---

## 2. Log Gateway / Collector

Purpose:

Receive logs from services.

Functions:

* authentication
* rate limiting
* batching
* compression

Example tools:

```
Fluentd
FluentBit
Vector
```

---

## 3. Kafka (Buffer Layer)

Kafka acts as the **central ingestion pipeline**.

Benefits:

```
decouples producers and consumers
absorbs traffic spikes
durable storage
replay capability
```

Topics example:

```
logs-topic
logs-error-topic
logs-security-topic
```

---

## 4. Log Processing Layer

Consumers process logs:

Responsibilities:

* enrichment
* parsing
* filtering
* indexing

Example processing tasks:

```
add service metadata
extract fields
detect anomalies
```

Frameworks used:

```
Flink
Spark Streaming
Kafka Streams
```

---

## 5. Storage Layer

Logs stored in **two tiers**.

### Hot Storage

Used for recent logs.

Technology:

```
Elasticsearch
OpenSearch
ClickHouse
```

Retention:

```
7 days
```

---

### Cold Storage

Used for archival.

Technology:

```
S3
GCS
HDFS
```

Retention:

```
90 days+
```

---

# 7. Log Query System

Query flow:

```
Dashboard
   |
Query API
   |
Elasticsearch
```

Search example:

```
service=payment-service
level=ERROR
time=last 30 minutes
```

---

# 8. Dashboard and Monitoring

Visualization tools:

```
Kibana
Grafana
Datadog
```

Capabilities:

* service error rate
* log heatmaps
* anomaly detection

---

# 9. Deep Dives (~10 minutes)

## 9.1 Scaling Log Ingestion

We shard Kafka topics.

Example:

```
logs-topic
    |
  100 partitions
```

Multiple collectors write concurrently.

---

## 9.2 Log Batching

Instead of sending:

```
1 log request
```

SDK sends:

```
batch of 100 logs
```

Reduces network overhead.

---

## 9.3 Backpressure Handling

If Kafka slows down:

SDK behavior:

```
retry with exponential backoff
```

Collector behavior:

```
temporary buffering
```

---

## 9.4 Failure Handling

### Collector Failure

Solution:

```
Load balancer
multiple collectors
```

---

### Kafka Failure

Solution:

```
replication factor = 3
```

---

### Storage Failure

Solution:

```
replicated clusters
multi-AZ
```

---

## 9.5 Log Compression

Use:

```
gzip
snappy
lz4
```

Compression reduces:

```
storage cost
network bandwidth
```

---

## 9.6 Log Sampling

For extremely noisy logs.

Example:

```
debug logs sampled at 10%
```

Reduces load.

---

## 9.7 Multi-Tenant Isolation

Large organizations have many teams.

Solution:

```
separate Kafka topics
service-based indexing
quota enforcement
```

---

# 10. Security

Logs often contain sensitive data.

Protections:

```
PII filtering
encryption at rest
encryption in transit
role based access control
```

---

# 11. Industry Implementations

### ELK Stack

```
Logstash
Elasticsearch
Kibana
```

---

### EFK Stack (Kubernetes)

```
Fluentd
Elasticsearch
Kibana
```

---

### Datadog

Managed logging platform.

---

### Splunk

Enterprise log analytics platform.

---

# 12. Final Architecture

```
          Services
             |
         Logging SDK
             |
        Log Collectors
             |
            Kafka
             |
      Stream Processing
             |
   ------------------------
   |                      |
Hot Storage         Cold Storage
(Elastic)            (S3/HDFS)
   |
Query API
   |
Dashboard (Grafana/Kibana)
```

---

# 13. Staff-Level Engineering Considerations

A staff engineer focuses on:

### Reliability

System must never block application threads.

---

### Cost Control

Logs generate **petabytes of data**.

Strategies:

```
sampling
compression
tiered storage
```

---

### Developer Experience

Logging SDK must be:

```
easy to integrate
language agnostic
structured logging friendly
```

---

### Observability Integration

Logs must integrate with:

```
metrics
tracing
alerting
```

---

# Final Thought

The goal of a distributed logging system is not just storage.

It enables:

```
debugging
incident response
security monitoring
performance analysis
```

A well-designed logging framework becomes **the backbone of observability across the entire organization**.
