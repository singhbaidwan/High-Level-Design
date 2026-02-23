# CQRS

Source: extracted from `architects/interview-ready.md`

### 7.1 Definition and Overview

CQRS separates write (command) and read (query) models for independent optimization.

Core characteristics:

- Write model optimized for correctness
- Read model optimized for query performance
- Often combined with event-driven updates

### 7.2 Where it is used

- Read-heavy systems
- Complex reporting/dashboard requirements
- Systems with divergent write vs read needs

### 7.3 Delivery Framework

Functional requirements:

- Commands mutate canonical state
- Queries served from read projections
- Projection updates via events/CDC

Non-functional requirements:

- Scalability: read and write scale independently
- Availability: read model can stay up during write pressure
- Consistency: eventual consistency between models
- Latency: fast reads from denormalized views
- Security: separate authorization rules for commands vs queries
- Fault tolerance: projection rebuild/replay strategy required

Capacity considerations:

- Projection storage size and rebuild time
- Event replay throughput for catch-up

### 7.4 Core entities

- Command entities (normalized, transactional)
- Query entities (denormalized views)
- Projection offsets/versioning

### 7.5 API and interface

- Command endpoints for writes
- Query endpoints for reads
- Internal projection pipeline updates read side

### 7.6 Diagram and flow

![CQRS A](../images/summary-21.png)
![CQRS B](../images/summary-22.png)

Flow:

1. Write to command model
2. Publish/update event
3. Projection updates read model
4. Query endpoint serves fast read

### 7.7 Pros

- Independent tuning of read/write paths
- High read performance

### 7.8 Cons

- More infrastructure and data models
- Eventual consistency handling

### 7.9 How we overcome limitations

- Clear staleness expectations in APIs
- Automated projection rebuilds
- Monitoring for projection lag

### 7.10 When to choose

Choose CQRS when read workload dominates and query patterns differ sharply from writes.

Conclusion: CQRS improves performance and scalability at the cost of operational complexity.

