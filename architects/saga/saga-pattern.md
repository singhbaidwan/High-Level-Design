# Saga Pattern

Source: extracted from `architects/interview-ready.md`

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

![Saga Orchestration A](../images/summary-17.png)
![Saga vs 2PC](../images/summary-19.png)

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

