# Hexagonal (Ports and Adapters)

Source: extracted from `architects/interview-ready.md`

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

![Hexagonal Example A](../images/summary-09.png)
![Hexagonal Example B](../images/summary-10.jpg)

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

