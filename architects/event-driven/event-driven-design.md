# Event-Driven Design

Source: extracted from `architects/interview-ready.md`

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

![Kafka/Event Flow A](../images/summary-13.png)
![Kafka/Event Flow B](../images/summary-14.png)

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

