# Scalable Withdrawal System: Interview-Ready Architecture Notes

This document uses a consistent delivery framework to explain major architecture patterns for system design interviews.

## Architecture Roadmap

1. Monolith to Microservices
2. Layered Architecture
3. Hexagonal (Ports and Adapters)
4. Event-Driven Design
5. Saga Pattern
6. Outbox Pattern
7. CQRS
8. Cell-Based Architecture
9. Active-Active vs Active-Passive
10. Observability-Driven Architecture
11. Platform Tenant Architecture

## Section Files

- `3. Hexagonal (Ports and Adapters)` -> `architects/hexagonal/hexagonal-architecture.md`
- `4. Event-Driven Design` -> `architects/event-driven/event-driven-design.md`
- `5. Saga Pattern` -> `architects/saga/saga-pattern.md`
- `6. Outbox Pattern` -> `architects/outbox/outbox-pattern.md`
- `7. CQRS` -> `architects/cqrs/cqrs.md`
- `8. Cell-Based Architecture` -> `architects/cell-based/cell-based-architecture.md`
- `9. Active-Active vs Active-Passive` -> `architects/active-active-passive/active-active-vs-active-passive.md`
- `10. Observability-Driven Architecture` -> `architects/observability/observability-driven-architecture.md`
- `11. Platform Tenant Architecture` -> `architects/platform-tenant/platform-tenant-architecture.md`

## 1. Monolith to Microservices

### Why start with monolith

- Fastest path to launch
- Simple local development and deployment
- Strong transaction boundaries with one DB

### Why evolve to microservices

- Independent scaling
- Independent deployability
- Better domain ownership by teams

## 2. Layered Architecture (Inside Services)

Typical structure:

- Controller layer
- Service layer
- Repository layer
- Database layer

Why it matters:

- Clear separation of concerns
- Better maintainability and testability

## 3. Hexagonal (Ports and Adapters)

### 3.1 Definition and Overview

Hexagonal architecture isolates core business logic from infrastructure.

Core characteristics:

- Domain at center
- Ports define contracts
- Adapters implement external integrations
- Business logic independent from DB, Kafka, or payment providers

Typical stack examples:

- Java Spring Boot with interfaces + adapter classes
- .NET with application/core/infrastructure projects
- Node.js with use-case layer + gateway adapters

### 3.2 Where it is used

- Payment systems with changing providers
- Systems integrating multiple external dependencies
- Platforms needing strict testability and boundary control

### 3.3 Delivery Framework

Functional requirements:

- User actions invoke domain use-cases
- CRUD and business rules stay in domain/application layers
- Auth context is passed into use-cases via input ports

Non-functional requirements:

- Scalability: neutral by itself, depends on deployment model
- Availability: improved by isolating flaky dependencies behind adapters
- Consistency: domain rules remain explicit and testable
- Latency: one more abstraction layer, usually low overhead
- Security: easier policy placement at adapters and application layer
- Fault tolerance: adapter-level retries/circuit breakers are easier

Capacity considerations:

- Scale app instances horizontally
- External dependencies can be swapped or optimized per adapter
- DB pressure still exists if write-heavy workloads are centralized

### 3.4 Core entities

- Domain entities remain framework-agnostic
- Repositories represented as ports
- Persistence models are adapter concerns

### 3.5 API and interface

- External REST/gRPC enters through inbound adapters
- Internal flow is in-process through interfaces (ports)
- Outbound calls use adapter implementations

### 3.6 Diagram and flow

![Hexagonal Example A](images/summary-09.png)
![Hexagonal Example B](images/summary-10.jpg)

Request flow:

1. Controller adapter receives request
2. Input port invokes domain use-case
3. Domain calls outbound ports
4. Outbound adapters interact with DB/Kafka/gateway
5. Response mapped back to API model

### 3.7 Pros

- Strong boundary control
- Easier testing with mocks/fakes
- Lower vendor lock-in

### 3.8 Cons

- More abstractions and boilerplate
- Requires design discipline

### 3.9 How we overcome limitations

- Standard port naming conventions
- Template-driven module structure
- DDD alignment for clear boundaries

### 3.10 When to choose

Choose Hexagonal when integration complexity is high and domain correctness matters more than short-term coding speed.

Conclusion: Hexagonal improves long-term maintainability and change safety.

## 4. Event-Driven Design

### 4.1 Definition and Overview

Event-driven architecture uses asynchronous events to communicate state changes between services.

Core characteristics:

- Producers publish events
- Brokers route/store events
- Consumers react independently
- Loose coupling between components

Typical stack examples:

- Kafka + Spring Boot
- SNS/SQS or EventBridge + Lambda/ECS
- RabbitMQ + workers

### 4.2 Where it is used

- High-throughput transactional systems
- Workflows requiring async processing
- Integrations across many bounded contexts

### 4.3 Delivery Framework

Functional requirements:

- User action creates command and domain event
- Downstream services consume and process independently
- Supports decoupled fraud, notifications, reconciliation

Non-functional requirements:

- Scalability: strong through partitioned consumption
- Availability: consumers can recover via replay
- Consistency: eventual consistency by default
- Latency: async improves tail latency for user path, but adds propagation delay
- Security: event schemas and topic ACLs required
- Fault tolerance: retries, DLQ, replay handling are essential

Capacity considerations:

- Partition count drives parallelism
- Consumer lag must be monitored
- Storage retention and replay cost must be planned

### 4.4 Core entities

- Domain events (`WithdrawalCreated`, `WithdrawalSettled`)
- Event keys for ordering/idempotency
- Consumer state/checkpoint metadata

### 4.5 API and interface

- Write API often synchronous
- Side effects handled asynchronously via event topics
- Internal network calls replaced by event subscriptions where possible

### 4.6 Diagram and flow

![Kafka/Event Flow A](images/summary-13.png)
![Kafka/Event Flow B](images/summary-14.png)

Flow:

1. Service writes transaction
2. Event published to topic
3. Consumers process independently
4. Failures retried or sent to DLQ

### 4.7 Pros

- Loose coupling
- Better elasticity
- Recovery via replay

### 4.8 Cons

- Eventual consistency complexity
- Harder debugging and tracing
- Schema evolution challenges

### 4.9 How we overcome limitations

- Schema registry and compatibility policies
- Idempotent consumers
- Central observability for lag/failures

### 4.10 When to choose

Choose event-driven when throughput, decoupling, and async workflows are core requirements.

Conclusion: Event-driven design scales distributed workflows but needs operational discipline.

## 5. Saga Pattern

### 5.1 Definition and Overview

Saga manages distributed transactions as a sequence of local transactions with compensation.

Core characteristics:

- No global ACID transaction across services
- Forward steps + compensating steps
- Orchestrated or choreographed execution

### 5.2 Where it is used

- Payments and order flows
- Booking and reservation systems
- Multi-service business workflows

### 5.3 Delivery Framework

Functional requirements:

- Execute stepwise business workflow
- Roll back business effect through compensations on failure

Non-functional requirements:

- Scalability: better than 2PC for distributed systems
- Availability: partial failures can be recovered via compensation
- Consistency: eventual business consistency
- Latency: longer end-to-end flow vs single transaction
- Security: step-level auth and audit needed
- Fault tolerance: state machine durability is critical

Capacity considerations:

- Orchestrator throughput and state storage capacity
- Retry storms must be controlled
- Compensation volume increases during downstream outages

### 5.4 Core entities

- Saga state (`PENDING`, `COMPLETED`, `COMPENSATING`)
- Step metadata and correlation IDs
- Compensating actions per step

### 5.5 API and interface

- Initiating API starts saga
- Service-to-service operations via commands/events
- State progression persisted for resume/retry

### 5.6 Diagram and flow

![Saga Orchestration A](images/summary-17.png)
![Saga vs 2PC](images/summary-19.png)

Flow:

1. Debit ledger
2. Execute gateway transfer
3. On failure, compensate ledger

### 5.7 Pros

- Practical distributed consistency model
- Works at high scale

### 5.8 Cons

- Complex error paths
- Compensation design can be hard

### 5.9 How we overcome limitations

- Explicit saga state machine
- Idempotent commands and compensations
- Timeouts and dead-letter handling

### 5.10 When to choose

Choose Saga when one business transaction spans multiple services and strict 2PC is impractical.

Conclusion: Saga is the standard pattern for correctness in distributed workflows.

## 6. Outbox Pattern

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

![Saga Compensation Example](images/summary-20.png)

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

## 7. CQRS

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

![CQRS A](images/summary-21.png)
![CQRS B](images/summary-22.png)

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

## 8. Cell-Based Architecture

### 8.1 Definition and Overview

Cell-based architecture partitions users/tenants into isolated, repeatable cells.

Core characteristics:

- Each cell has its own app + data slice
- Failures stay within a cell boundary
- Cells are horizontally replicable units

### 8.2 Where it is used

- Large-scale multi-tenant platforms
- Financial systems requiring blast-radius control
- Global systems with regional partitioning

### 8.3 Delivery Framework

Functional requirements:

- Route user to owning cell
- Serve all core workflows within that cell

Non-functional requirements:

- Scalability: add cells to increase capacity
- Availability: cell failure impacts subset, not all users
- Consistency: strong within a cell, cross-cell logic is constrained
- Latency: improved via locality and smaller working sets
- Security: tenant and cell isolation improves containment
- Fault tolerance: operational failures are localized

Capacity considerations:

- Cell count planning by users/traffic
- Rebalancing and migration cost across cells

### 8.4 Core entities

- Cell routing map (`tenant -> cell`)
- Per-cell service and data ownership
- Control-plane metadata for provisioning

### 8.5 API and interface

- Edge layer resolves cell routing
- Requests forwarded to cell-local APIs
- Prefer no cross-cell synchronous dependency

### 8.6 Diagram and flow

![Cell Architecture A](images/summary-25.jpg)
![Cell Architecture B](images/summary-26.png)

Flow:

1. Identify tenant/user
2. Map to cell
3. Process request in that cell only

### 8.7 Pros

- Strong blast-radius reduction
- Predictable horizontal scale model

### 8.8 Cons

- Operational overhead per cell
- Cross-cell analytics/operations complexity

### 8.9 How we overcome limitations

- Strong control-plane automation
- Standardized cell templates
- Controlled migration tooling

### 8.10 When to choose

Choose cell-based design when reliability isolation is a top requirement at high scale.

Conclusion: Cells trade operational simplicity for much stronger failure containment.

## 9. Active-Active vs Active-Passive

### 9.1 Definition and Overview

Active-passive runs traffic primarily in one region while another is standby. Active-active serves traffic from multiple regions simultaneously.

Core characteristics:

- Active-passive: simpler failover model
- Active-active: better availability, harder consistency model

### 9.2 Where it is used

- DR-sensitive systems
- Global products with multi-region users
- Financial systems with strict resilience needs

### 9.3 Delivery Framework

Functional requirements:

- Keep service available during regional failures
- Route traffic based on health and ownership strategy

Non-functional requirements:

- Scalability: active-active uses global capacity better
- Availability: active-active usually offers lower RTO/RPO
- Consistency: active-active adds conflict-resolution complexity
- Latency: active-active improves latency via local serving
- Security: replicated controls and key management across regions
- Fault tolerance: region outages handled by traffic failover

Capacity considerations:

- Active-passive keeps spare headroom in standby
- Active-active requires partition ownership or conflict resolution

### 9.4 Core entities

- Region ownership metadata
- Replication logs/checkpoints
- Conflict policy definitions

### 9.5 API and interface

- Global traffic manager directs requests
- Writes should follow ownership rules
- Read routing depends on freshness requirements

### 9.6 Diagram and flow

![Consolidated C](images/summary-31.png)

Flow:

1. Client routed to region
2. Region serves request
3. Replication/failover policy maintains continuity

### 9.7 Pros

- Strong resiliency options
- Better global experience in active-active

### 9.8 Cons

- Active-active conflict complexity
- Higher ops and data replication overhead

### 9.9 How we overcome limitations

- Regional ownership (single-writer per entity)
- Deterministic failover runbooks
- Replication health SLOs

### 9.10 When to choose

- Choose active-passive for simpler operations and clear DR.
- Choose active-active for strict uptime and low-latency global traffic when org maturity is high.

Conclusion: Use the simplest multi-region model that satisfies resilience targets.

## 10. Observability-Driven Architecture

### 10.1 Definition and Overview

Observability-driven architecture designs systems so failures are quickly detectable, explainable, and recoverable.

Core characteristics:

- Metrics, logs, traces as first-class design outputs
- SLO-based monitoring and alerting
- Correlation IDs across services/events

### 10.2 Where it is used

- Any production distributed system
- High-SLA products
- Regulated systems requiring auditability

### 10.3 Delivery Framework

Functional requirements:

- Every critical workflow emits telemetry
- Operators can trace request-to-event-to-database path

Non-functional requirements:

- Scalability: telemetry pipelines must scale with workload
- Availability: alerting and dashboards must remain reliable
- Consistency: timestamps and trace context must be consistent
- Latency: instrumentation overhead must be controlled
- Security: redact PII and secure observability data stores
- Fault tolerance: degraded telemetry should not break business path

Capacity considerations:

- Log volume and retention cost
- Metrics cardinality control
- Trace sampling strategies

### 10.4 Core entities

- Trace IDs/span IDs
- Metric dimensions and service-level indicators
- Structured log schemas

### 10.5 API and interface

- Instrument HTTP, DB, queue, and external calls
- Emit standardized events to observability backend

### 10.6 Diagram and flow

![Consolidated D](images/summary-32.png)

Flow:

1. Request enters service with correlation ID
2. Metrics/logs/traces emitted per step
3. Alerting evaluates SLO thresholds
4. On-call uses traces and logs for root cause

### 10.7 Pros

- Faster incident detection and resolution
- Better change safety and capacity planning

### 10.8 Cons

- Tooling and storage cost
- Noise if instrumentation standards are weak

### 10.9 How we overcome limitations

- Golden signals and SLO-first dashboards
- Log schema standards and PII controls
- Alert tuning with error-budget policies

### 10.10 When to choose

Choose observability-driven architecture from day one for distributed systems; retrofitting later is expensive.

Conclusion: Observability is a core architecture requirement, not an add-on.

## Interview-Ready Wrap-Up

A strong closing narrative is:

- Start with simple boundaries and clear layering.
- Apply hexagonal design for domain isolation.
- Use event-driven patterns for decoupling and elasticity.
- Guarantee reliability with Saga + Outbox + idempotency.
- Split read/write concerns with CQRS.
- Reduce blast radius with cell-based partitioning.
- Pick multi-region model based on maturity and consistency needs.
- Design observability as a first-class system capability.

## Notes

- All diagrams are referenced locally from `architects/images`.
