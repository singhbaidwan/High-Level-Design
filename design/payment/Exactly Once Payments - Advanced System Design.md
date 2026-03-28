# 💳 Exactly-Once Payments in Distributed Systems (Advanced Guide)

---

# 🚀 1. Problem Statement

In distributed systems, especially payment systems, ensuring that a transaction executes **exactly once** is extremely challenging due to:

* Network failures
* Timeouts
* Retries
* Partial failures

### ❗ Core Question

> How do you guarantee that a payment is processed exactly once?

---

# ⚠️ 2. Why Exactly-Once is Hard

Distributed systems inherently provide only:

| Guarantee     | Issue                      |
| ------------- | -------------------------- |
| At-most-once  | Requests can be lost       |
| At-least-once | Requests can be duplicated |

👉 Exactly-once must be **engineered at the application layer**.

---

# 🧠 3. Core Insight

> A retry is indistinguishable from a new request unless explicitly designed otherwise.

---

# 🧩 4. Idempotency Keys (Baseline Solution)

## ✅ Concept

Each logical operation gets a unique key.

### Flow:

1. Client generates UUID
2. Sends request with idempotency key
3. Server stores key + result
4. Retry → return stored result

---

## 📦 Example

```http
POST /payments
Idempotency-Key: abc123
```

---

## ✅ Pros

* Simple
* Widely used
* Prevents duplicate execution

## ❌ Cons

* Fails if crash happens before result is stored

---

# 🔄 5. Idempotency State Machine (Robust Solution)

## 💡 Upgrade

Instead of key → result:

Use key → state machine

---

## 🔁 States

| State       | Meaning                |
| ----------- | ---------------------- |
| IN_PROGRESS | Execution ongoing      |
| COMPLETED   | Success stored         |
| FAILED      | Optional failure state |

---

## 🧭 Flow

1. Insert record as IN_PROGRESS
2. Execute operation
3. Update to COMPLETED with result

---

## 🔁 Retry Handling

| State       | Action                   |
| ----------- | ------------------------ |
| COMPLETED   | Return cached result     |
| IN_PROGRESS | Return 409 / retry later |
| NOT FOUND   | Start new execution      |

---

## ⚙️ Concurrency Control

* Compare-And-Swap (CAS)
* Distributed locks (Redis, Zookeeper)

---

## ⚖️ Trade-offs

| Factor       | Impact                     |
| ------------ | -------------------------- |
| Latency      | Increased                  |
| Availability | Idempotency store critical |
| Complexity   | Higher                     |

---

# 💣 6. Partial Failure Problem

## Scenario:

1. Payment charged ✅
2. Inventory update ❌
3. Email not sent ❌

System becomes inconsistent.

---

# 🔗 7. Distributed Transactions Problem

## ❌ Why Not 2-Phase Commit?

* High latency
* Blocking
* Poor scalability
* Fragile under failures

---

# 🧵 8. Saga Pattern (Distributed Coordination)

## 💡 Concept

Break transaction into multiple steps with compensations.

---

## 🧭 Example

| Step | Action           | Compensation  |
| ---- | ---------------- | ------------- |
| 1    | Charge card      | Refund        |
| 2    | Update inventory | Restore stock |
| 3    | Send email       | Ignore        |

---

## 🔁 Flow

* Execute steps sequentially
* On failure → execute compensating actions

---

## ⚠️ Properties

| Property    | Value                 |
| ----------- | --------------------- |
| Atomicity   | ❌ Not guaranteed      |
| Consistency | Eventually consistent |
| Reliability | High                  |

---

## 🔄 Saga Types

### 1. Orchestrated Saga

* Central coordinator

### 2. Choreographed Saga

* Event-driven

---

# 📦 9. Outbox Pattern (Critical for Reliability)

## 💡 Problem

Dual write problem:

* DB write succeeds
* Event publish fails

---

## ✅ Solution

* Write event to "outbox table" inside same DB transaction
* Background worker publishes events

---

## Benefits

* Guarantees event delivery
* Avoids inconsistency

---

# 🔁 10. Retry & Idempotency Everywhere

## Rule:

Every operation must be retry-safe

---

## Examples

* Payment → idempotent
* Refund → idempotent
* Inventory restore → idempotent

---

# 🔥 11. Cross-Service Idempotency (Advanced)

## Problem

Retries at saga level can cause duplicate compensations.

---

## Solution

* Each service maintains its own idempotency keys
* Global transaction ID propagated across services

---

# 🧠 12. Advanced Failure Handling

## Techniques

* Dead Letter Queues (DLQ)
* Circuit breakers
* Timeouts + retries with backoff
* Monitoring + alerting

---

# ⚖️ 13. Trade-offs Summary

| Approach        | Pros                         | Cons             |
| --------------- | ---------------------------- | ---------------- |
| Idempotency Key | Simple                       | Weak under crash |
| State Machine   | Strong correctness           | Latency          |
| Saga            | Handles distributed failures | Complex          |
| Outbox          | Reliable messaging           | Extra infra      |

---

# 🧠 14. Interview Mental Model

### Case 1: No side effects

→ No idempotency needed

### Case 2: Single service

→ Idempotency + state machine

### Case 3: Multiple services

→ Saga + Outbox + Idempotency

---

# 💬 15. Golden Statement

> Exactly-once semantics in distributed systems must be implemented at the application layer using idempotency.

---

# 🏗️ 16. Real-World Architecture (High-Level)

Client → API Gateway → Payment Service →

* Idempotency Store (Redis/DB)
* Payment Processor
* Outbox Table

Then:

* Event Bus (Kafka)
* Downstream services (Inventory, Email)

---

# 📌 17. Final Takeaways

* Failures are normal
* Retries are unavoidable
* Idempotency is mandatory
* Distributed consistency requires sagas

---

# 🚀 18. Next Steps (Practice)

* Implement idempotency service in Redis
* Build saga with Kafka
* Add outbox pattern
* Simulate failures

---

# 🏁 Conclusion

Exactly-once execution is not a built-in guarantee—it is a carefully engineered property combining:

* Idempotency
* State management
* Distributed coordination

Mastering these concepts is essential for designing reliable large-scale systems.
