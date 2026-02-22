# Scalable Withdrawal System: Architecture Summary

This document summarizes how to evolve a withdrawal platform from early-stage simplicity to large-scale reliability.

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

## 1. Monolith to Microservices

### Why start with a monolith

- Fastest path to production
- Simpler deployments and debugging
- Strong local transactions

![Monolith Decomposition](images/summary-01.jpg)
![Monolith Context](images/summary-02.jpg)
![Client Server Reference](images/summary-03.svg)
![Shared DB Pattern](images/summary-04.png)

### Why evolve to microservices

- Independent scaling by domain
- Independent deployments
- Better team ownership boundaries

![Microservices Example A](images/summary-05.png)
![Microservices Example B](images/summary-06.png)
![Microservices Example C](images/summary-07.png)
![Microservices Example D](images/summary-08.png)

Typical service split for withdrawals:

- Withdrawal Service
- Ledger Service
- Fraud Service
- Gateway Adapter Service
- Reconciliation Service

## 2. Layered Architecture (Inside Each Service)

Use clear separation inside each service:

- Controller: request/response handling
- Service: business rules
- Repository: data access

```java
@RestController
class WithdrawalController {
  @PostMapping("/withdraw")
  public Response withdraw(@RequestBody Req req) {
    return withdrawalService.process(req);
  }
}
```

Benefits:

- Better testability
- Lower accidental coupling
- Easier maintenance

## 3. Hexagonal Architecture (Ports and Adapters)

Goal: isolate domain logic from infrastructure details.

![Hexagonal Example A](images/summary-09.png)
![Hexagonal Example B](images/summary-10.jpg)
![Hexagonal Example C](images/summary-11.png)
![Hexagonal Example D](images/summary-12.jpg)

Suggested structure:

- Domain core: `WithdrawalProcessor`, `FraudPolicy`
- Ports: `PaymentGatewayPort`, `EventPublisherPort`
- Adapters: `KafkaAdapter`, `PostgresAdapter`, `GatewayAdapter`

## 4. Event-Driven Architecture

Use asynchronous messaging to reduce tight runtime coupling.

![Kafka/Event Flow A](images/summary-13.png)
![Kafka/Event Flow B](images/summary-14.png)
![Kafka/Event Flow C](images/summary-15.png)
![Kafka/Event Flow D](images/summary-16.png)

When this helps most:

- Burst traffic and variable downstream latency
- Long-running fraud or reconciliation workflows
- Replay and recovery requirements

```java
kafkaTemplate.send("withdrawal-topic", event);

@KafkaListener(topics = "withdrawal-topic")
public void consume(Event e) {
  process(e);
}
```

## 5. Saga Pattern (Distributed Consistency)

Use sagas when one business action spans multiple services.

![Saga Orchestration A](images/summary-17.png)
![Saga Orchestration B](images/summary-18.png)
![Saga vs 2PC](images/summary-19.png)
![Saga Compensation Example](images/summary-20.png)

Canonical flow:

1. Reserve/debit ledger
2. Execute gateway transfer
3. On failure, run compensation (credit back)

```java
if (paymentFailed) {
  ledger.credit(userId, amount);
}
```

## 6. Outbox Pattern (Reliable Publish)

Problem: DB commit and event publish are separate systems.

Solution:

- Write business state + outbox record in one DB transaction
- Relay process publishes outbox to Kafka
- Relay marks outbox record as sent

Why it matters:

- No lost events after successful DB commits
- Predictable retry behavior

## 7. CQRS (Command Query Separation)

Separate write and read models when both have very different needs.

![CQRS A](images/summary-21.png)
![CQRS B](images/summary-22.png)
![CQRS C](images/summary-23.png)
![CQRS D](images/summary-24.webp)

Pattern:

- Write side: ACID ledger updates
- Read side: materialized views for fast balance/history queries

## 8. Cell-Based Architecture

Partition users/tenants into isolated cells to reduce blast radius.

![Cell Architecture A](images/summary-25.jpg)
![Cell Architecture B](images/summary-26.png)
![Cell Architecture C](images/summary-27.png)
![Cell Architecture D](images/summary-28.jpg)

Example routing:

```text
cell = hash(userId) % N
```

Each cell can own:

- Service stack
- Data stores
- Event infrastructure

## 9. Active-Active vs Active-Passive

### Active-passive

- Simpler operations
- Slower failover

### Active-active

- Better availability
- Conflict complexity if same entity is writable in multiple regions

For financial systems, prefer clear ownership boundaries per user/account/ledger shard.

## 10. Observability-Driven Architecture

Design for diagnosis from day one.

Mandatory telemetry:

- Distributed tracing
- Structured logs with correlation IDs
- Service and infra metrics
- SLO-based alerting

Key metrics:

- `p99` end-to-end latency
- Error rate and error budget burn
- Kafka consumer lag
- DB lock wait time

## Consolidated Reference Diagrams

![Consolidated A](images/summary-29.png)
![Consolidated B](images/summary-30.svg)
![Consolidated C](images/summary-31.png)
![Consolidated D](images/summary-32.png)

## Interview-Ready Conclusion

A strong design narrative is:

- Start simple with monolith + clean layers.
- Extract bounded-domain microservices as scale and teams grow.
- Keep domain logic isolated with hexagonal boundaries.
- Use event-driven flows for decoupling and elasticity.
- Ensure consistency with saga + outbox + idempotency.
- Scale reads independently with CQRS.
- Limit failures with cell-based partitioning.
- Improve resilience and operations with observability-first engineering.

## Notes

- All external image links from the previous version were localized to `architects/images`.
