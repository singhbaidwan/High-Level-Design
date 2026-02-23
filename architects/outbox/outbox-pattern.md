# Outbox Pattern

Source: extracted from `architects/interview-ready.md`

### 6.1 Definition and Overview

Outbox ensures reliable event publication by storing events in DB within the same transaction as business updates.

Core characteristics:

- Business row + outbox row in one commit
- Relay publishes outbox asynchronously
- Retries safe with idempotency

### 6.2 Where it is used

- Systems requiring guaranteed event delivery after commit
- Financial and audit-heavy workflows

### 6.3 Delivery Framework

Functional requirements:

- Persist business change
- Persist integration event atomically
- Publish event and mark as sent

Non-functional requirements:

- Scalability: relay workers can scale horizontally
- Availability: event not lost if broker temporarily down
- Consistency: prevents DB-commit/event-publish mismatch
- Latency: slight delay between commit and publish
- Security: outbox data must be protected and audited
- Fault tolerance: at-least-once delivery requires idempotent consumers

Capacity considerations:

- Outbox table growth and cleanup policy
- Relay throughput and retry backoff tuning

### 6.4 Core entities

- Outbox record (`id`, `event_type`, `payload`, `status`, `created_at`)
- Relay checkpoint/lease metadata

### 6.5 API and interface

- Normal synchronous API write path
- Async relay to broker from outbox store

### 6.6 Diagram and flow

![Saga Compensation Example](../images/summary-20.png)

Flow:

1. Transaction writes business + outbox
2. Relay reads unsent events
3. Relay publishes to broker
4. Relay marks event published

### 6.7 Pros

- Strong reliability between DB and broker
- Cleaner recovery behavior

### 6.8 Cons

- Extra table and relay operations
- Requires cleanup and monitoring

### 6.9 How we overcome limitations

- Partitioned outbox tables
- Batched relay publishing
- Idempotent consumer contracts

### 6.10 When to choose

Choose Outbox whenever domain state change must reliably emit an event.

Conclusion: Outbox is a baseline correctness pattern in event-driven systems.

