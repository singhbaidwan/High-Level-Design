I'll review this exactly the way a FAANG Staff Engineer (L6/L7 interviewer) would during a System Design interview.

---

# Interview Evaluation First

If this were an **SDE II (L5)** interview:

**Verdict:** Hire / Lean Hire

Why:

* Requirements are captured.
* Event-driven architecture is used.
* Kafka introduced appropriately.
* Registration and notification responsibilities are separated.
* Delivery guarantees considered.
* Scalability concerns mentioned.

---

If this were an **SDE III (L6)** interview:

**Verdict:** Lean No Hire

Why:

* Architecture is mostly component-level.
* Missing deep discussion of:

  * failure handling
  * multi-region strategy
  * idempotency
  * webhook retries
  * security model
  * backpressure
  * tenant isolation
  * observability
  * DLQ strategy
  * exactly-once vs at-least-once tradeoffs
  * operational concerns

An L6 candidate is expected to proactively address these.

---

# What I See In Your Design

You are building a **Webhook Notification Platform**.

Think of something similar to:

* Stripe Webhooks
* GitHub Webhooks
* Slack Event Subscriptions

---

# Functional Requirements

You captured:

### Register webhook

```http
POST /webhooks
```

Organization subscribes to:

```json
{
  "event_type":"payment_completed",
  "webhook_url":"..."
}
```

Good.

---

### Trigger event

Internal services publish:

```http
POST /events
```

Example:

```json
{
  "organization_id":"123",
  "event_type":"payment_completed",
  "payload":{}
}
```

Good.

---

# High Level Flow

Current flow:

```text
Internal Service
      |
      v
 Kafka Topic
      |
      v
 Notification Service
      |
      +------> Webhook URL

      +------> Logs

      +------> Subscription DB
```

This is fundamentally correct.

---

# Detailed Component Review

---

## 1. Registration Service

Purpose:

```text
Create Subscription
Update Subscription
Delete Subscription
```

Stores:

```text
organization_id
event_type
webhook_url
secret_key
```

Good separation.

---

### Missing

Webhook verification.

Example:

When customer registers:

```json
{
   "url":"https://my-system.com/webhook"
}
```

You should verify ownership.

Common pattern:

```text
Send challenge token

POST /webhook

{
   "challenge":"abc123"
}
```

Customer returns:

```json
{
   "challenge":"abc123"
}
```

Only then activate subscription.

---

# 2. PostgreSQL

Current schema:

```text
Organizations

Subscriptions
```

Good.

---

### Missing indexes

Need:

```sql
CREATE INDEX idx_org_event
ON subscriptions(organization_id,event_type);
```

Otherwise lookup becomes expensive.

---

### Missing subscription status

```sql
status
--------
ACTIVE
PAUSED
DISABLED
VERIFYING
```

Important.

---

# 3. Kafka

This is where most interview signal comes from.

You mention:

```text
organization_id topic
```

I would NOT do that.

---

## Why?

Imagine:

```text
100,000 organizations
```

Then:

```text
100,000 Kafka topics
```

This becomes operationally painful.

---

### Better

Use few topics.

Example:

```text
events
```

Partition key:

```text
organization_id
```

Kafka hashes:

```text
organization_id -> partition
```

Benefits:

* ordering per tenant
* manageable topic count
* scalable

---

### Partition Count

Suppose:

```text
50K events/sec
```

and

```text
1000 events/sec/partition
```

Need:

```text
50 partitions
```

Add headroom:

```text
100 partitions
```

---

# 4. Notification Service

This is your core component.

Current responsibilities:

```text
consume kafka
lookup subscription
sign payload
send webhook
```

Correct.

---

# Biggest Missing Piece

Retries.

Webhook consumers fail constantly.

Examples:

```text
429
500
502
503
timeout
```

must retry.

---

### Retry Strategy

```text
Attempt 1 -> immediately
Attempt 2 -> 30 sec
Attempt 3 -> 2 min
Attempt 4 -> 10 min
Attempt 5 -> 1 hour
```

Exponential backoff.

---

### Dead Letter Queue

After max retries:

```text
webhook-dlq
```

Store:

```json
{
  "event_id":"123",
  "organization_id":"xyz",
  "reason":"503"
}
```

Critical for production.

---

# 5. Logging

You use Cassandra.

Good choice.

Reason:

```text
write-heavy
append-only
```

Perfect fit.

---

Store:

```text
event_id
delivery_attempt
status
latency
response_code
timestamp
```

Not only payload.

---

# Security Review

This is the most important part.

You wrote:

```text
secret_key
```

Let's discuss.

---

# Is Storing Secret Key A Good Idea?

Answer:

### Yes

But NOT in plaintext.

---

Bad:

```sql
secret_key = "my-secret"
```

---

Better:

```text
Generate per subscription secret
Encrypt before storage
```

Store:

```text
encrypted_secret
```

---

# Production Design

Use:

```text
AWS KMS
Google Cloud KMS
Hashicorp Vault
```

Flow:

```text
Generate secret

Encrypt using KMS

Store encrypted blob

Decrypt only during webhook signing
```

---

# Why Not Hash?

People often suggest:

```text
SHA256(secret)
```

Wrong.

Reason:

Webhook signing requires original secret.

Need:

```text
HMAC(payload, secret)
```

Therefore decryption is required.

---

# Recommended Signature Scheme

Like Stripe.

Headers:

```http
X-Signature: HMAC_SHA256(...)
X-Timestamp: 17111111
```

Signature:

```text
HMAC(secret,
      timestamp + payload)
```

Prevents replay attacks.

---

# Missing Security Features

## Replay Protection

Current design vulnerable.

Attacker captures:

```http
POST webhook
```

and replays it.

---

Solution:

```http
X-Timestamp
```

Reject:

```text
older than 5 minutes
```

---

## IP Allowlisting

Publish:

```text
Webhook delivery IP ranges
```

Customers can whitelist.

---

## Mutual TLS (Enterprise)

Optional.

Useful for large customers.

---

# Major Scalability Problem

Current design:

```text
Notification Service
```

single logical component.

---

At scale:

```text
Kafka
   |
Worker Pool
   |
HTTP Delivery
```

Need:

```text
100+
workers
```

Horizontal scaling.

---

# Missing Idempotency

Webhook delivery is:

```text
At-Least-Once
```

not exactly-once.

Duplicates happen.

---

Every event should have:

```json
{
   "event_id":"uuid"
}
```

Customer deduplicates.

---

# Missing Event Store

Currently:

```text
Kafka -> Notification Service
```

What if consumer crashes?

---

Better:

```text
Kafka
   |
Event Store
   |
Notification Workers
```

Store:

```text
delivery_status
attempts
next_retry_time
```

Could be:

* DynamoDB
* Cassandra
* PostgreSQL

---

# Improved Production Architecture

```text
                 +-------------------+
                 | Registration API  |
                 +---------+---------+
                           |
                           v
                    PostgreSQL
                    (Subscriptions)

Internal Services
        |
        v
+------------------+
| Kafka Event Bus  |
+------------------+
        |
        v
+------------------+
| Event Processor  |
+------------------+
        |
        v
+------------------+
| Delivery Queue   |
| (Kafka/SQS)      |
+------------------+
        |
        v
+------------------+
| Webhook Workers  |
| 100-1000 nodes   |
+------------------+
        |
        +-------> Customer Webhook

        |
        +-------> Delivery Logs
                    (Cassandra)

        |
        +-------> DLQ

        |
        +-------> Metrics
```

---

# L6-Level Deep Dive Questions I Would Ask

As an interviewer, after seeing your diagram I would ask:

### Question 1

A customer endpoint starts returning:

```text
503
```

for 3 hours.

How do you prevent retry storms?

Expected:

* exponential backoff
* jitter
* circuit breaker

---

### Question 2

One enterprise customer generates:

```text
90%
```

of traffic.

How do you prevent tenant starvation?

Expected:

* tenant quotas
* rate limiting
* partitioning strategy

---

### Question 3

Kafka goes down.

What happens?

Expected:

* replication factor
* ISR
* multi-AZ deployment

---

### Question 4

How do you guarantee ordering?

Expected:

```text
partition by organization_id
```

---

### Question 5

How do you migrate 1 billion subscriptions?

Expected:

* online migration
* dual write
* backfill
* verification

---

# Final Assessment (FAANG Calibration)

| Area                   | Rating       |
| ---------------------- | ------------ |
| Requirements           | Strong Hire  |
| API Design             | Hire         |
| Data Model             | Hire         |
| Event Architecture     | Hire         |
| Scalability            | Lean Hire    |
| Reliability            | Lean No Hire |
| Security               | Lean No Hire |
| Multi-Tenancy          | No Hire      |
| Operational Excellence | No Hire      |

### Overall

**For L5 (SDE II): Hire**

**For L6 (SDE III): Downlevel to L5**

Reason: The foundation is good, but an L6 answer would need substantially more depth around reliability, security, tenant isolation, operational excellence, and failure-mode analysis.

The single biggest improvement would be introducing a dedicated delivery pipeline with retries, DLQs, idempotency, encrypted webhook secrets (via KMS/Vault), tenant-aware rate limiting, and comprehensive observability. That would move this design much closer to a strong L6-level webhook platform design.
