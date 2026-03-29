How does a digital wallet work? When users add money to their wallet, transfer funds to other users, or pay merchants, how does the system manage wallet balances atomically? How do you design a system that handles millions of wallet transactions while ensuring balance accuracy, supporting multiple top-up methods, and maintaining compliance?

Concepts:
Digital Wallet, Balance Management, Atomic Transactions, Top-Up Mechanisms, P2P Transfers, Merchant Payments, Wallet-to-Bank Transfers, Distributed Locking, ACID Transactions, Wallet Compliance

Full story for non-members | E-Books on Java/Microservices/Springboot | Whatsapp Group| Youtube | LinkedIn

Press enter or click to view image in full size

A Real-World Problem
Aadvik (Interviewer): “Sara, imagine you’re building a digital wallet system like Paytm or PhonePe wallet. Users can add money to their wallet, transfer funds to other users, pay merchants, and withdraw money to their bank accounts. The system needs to manage wallet balances atomically, support multiple top-up methods, and handle millions of transactions. How do you design this system?”

Sara (Candidate): [Thoughtful pause] “This is a complex financial system with critical balance management requirements. A digital wallet involves several key components:

Balance Management: Atomic credit/debit operations with ACID guarantees
Top-Up Mechanisms: Support bank transfers, cards, UPI for adding money
P2P Transfers: Transfer funds between wallet users
Merchant Payments: Pay merchants from wallet balance
Wallet-to-Bank Transfers: Withdraw money to bank accounts
Transaction Ledger: Maintain complete transaction history
Compliance: Wallet limits, KYC requirements, regulatory compliance
Distributed Locking: Prevent race conditions in balance updates”
Aadvik: “Exactly. And here’s what makes it interesting: Digital wallets process billions of transactions globally. A typical wallet system handles 50,000+ transactions per second during peak times. How do you handle this scale while ensuring balance accuracy and atomicity?”

Sara: “The scale is significant, and balance accuracy is critical. We need a highly available, distributed architecture that can handle high transaction volumes while maintaining ACID guarantees for balance operations. Let me start with some clarifying questions to understand the requirements better.”

Aadvik: “Great! Ask away.”

Sara: “I have several questions:

Scale and Usage:

What’s the expected transaction volume? Peak TPS?
How many wallet users are we supporting?
How many merchants accept wallet payments?
What’s the average transaction amount?
Functional Requirements:
5. What top-up methods do we support? (Bank transfer, Cards, UPI)
6. Do we support P2P transfers between wallets?
7. What wallet limits do we need to enforce?
8. Do we need KYC for wallet users?

Balance Management:
9. How do we ensure balance accuracy?
10. How do we handle concurrent balance updates?
11. What’s the transaction ledger structure?
12. How do we handle failed transactions?

Non-Functional Requirements:
13. What’s the acceptable latency for balance operations?
14. What’s the availability target?
15. What compliance requirements do we need to meet?”

Aadvik: “Excellent questions! Let me give you the requirements:”

Part 1: Requirements & Core Challenges
A. Functional Requirements
Wallet Operations:
Wallet account creation and KYC
Balance check and inquiry
Transaction history and statements
Wallet limits management
2. Top-Up Mechanisms:

Bank transfer (NEFT, IMPS, UPI)
Credit/Debit card
UPI direct
Cash deposit (via agents)
3. Transfer Operations:

P2P wallet transfers
Merchant payments from wallet
Wallet-to-bank transfers
Request money (collect from other users)
4. Balance Management:

Atomic credit operations
Atomic debit operations
Balance locking for concurrent transactions
Transaction ledger maintenance
5. Compliance & Limits:

Wallet balance limits
Transaction limits (daily, monthly)
KYC requirements
Regulatory compliance (RBI guidelines)
6. Security & Authentication:

User authentication (OTP, PIN, biometric)
Transaction authorization
Fraud detection
Audit logging
B. Non-Functional Requirements
Scale:
500 million transactions per month
Average: 190 TPS
Peak: 15,000+ TPS (during sales events)
100 million wallet users
1 million merchants
2. Latency:

Balance check: < 100ms (p95)
Transfer operation: < 500ms (p95)
Top-up operation: < 2 seconds (p95)
3. Availability:

99.99% uptime (allows ~4 minutes downtime/month)
Zero balance discrepancies
Automatic failover
4. Consistency:

ACID guarantees for balance operations
Zero balance loss or duplication
Transaction atomicity
5. Compliance:

RBI guidelines (for India)
KYC requirements
Transaction limits
Audit logging
C. Core Challenges
Atomic Balance Operations: Ensure balance updates are atomic and consistent
Concurrent Transactions: Handle multiple transactions on same wallet simultaneously
Distributed Locking: Prevent race conditions in distributed environment
Top-Up Integration: Integrate with multiple payment methods for top-up
High Volume: Handle 15,000+ TPS with sub-second latency
Balance Accuracy: Zero tolerance for balance discrepancies
Compliance: Meet regulatory requirements and wallet limits
Transaction Ledger: Maintain complete audit trail
Aadvik: “Also, let’s start by understanding the scale. Digital wallets handle massive volumes. How do you handle this?”

Sara: “The scale is significant. Let me break down the numbers:”

Part 2: Scale & Capacity Planning
A. Transaction Volume
Monthly Volume:

500 million transactions per month
Average: 190 TPS
Peak: 15,000+ TPS (during sales events: Diwali, festivals, etc.)
Transaction Distribution:

P2P transfers: 40% (200M/month)
Merchant payments: 35% (175M/month)
Top-up operations: 15% (75M/month)
Wallet-to-bank transfers: 10% (50M/month)
Peak Load Scenarios:

Festival sales: 20,000+ TPS
Salary day: 15,000+ TPS
Cashback events: 12,000+ TPS
B. Data Storage
Transaction Data:

500M transactions/month × 1KB per transaction = 500GB/month
Annual: 6TB/year
Retention: 7 years = 42TB
Wallet Data:

100M users × 2KB per wallet = 200GB
Balance data: Real-time, minimal storage
Transaction Ledger:

Complete audit trail for all transactions
500GB/month × 7 years = 42TB
Total Storage:

Transaction data: 42TB (7 years)
Wallet data: 200GB
Ledger data: 42TB
Total: ~84TB
C. Network Bandwidth
Peak TPS: 15,000

Request size: 2KB
Response size: 1KB
Total: 3KB per transaction
Peak Bandwidth:

15,000 TPS × 3KB = 45MB/s = 360Mbps
With overhead: ~450Mbps
Average Bandwidth:

190 TPS × 3KB = 570KB/s = 4.5Mbps
Part 3: High-Level Architecture
Sara: “Let me design the high-level architecture for the digital wallet system. I’ll start with a functional view, then show the detailed architecture.”

A. High-Level Functional Architecture
Press enter or click to view image in full size

Core Functional Components:

Wallet Service: Main balance management, coordinates all operations
Top-Up Service: Handles money addition via multiple payment methods
Transfer Service: P2P transfers and merchant payments
Withdrawal Service: Wallet-to-bank transfers
Payment Gateway: Integration with payment providers for top-up
B. Detailed System Architecture (Scaled)

C. Core Services
1. Wallet Service

Main balance management service
Coordinates all wallet operations
Handles balance checks, updates
Manages transaction ledger
2. Top-Up Service

Handles money addition to wallet
Integrates with payment gateway
Supports multiple top-up methods
Processes top-up callbacks
3. Transfer Service

P2P wallet transfers
Merchant payments from wallet
Request money functionality
Transfer validation and processing
4. Withdrawal Service

Wallet-to-bank transfers
Bank integration for withdrawals
Withdrawal processing and status tracking
5. Limit Service

Enforces wallet balance limits
Enforces transaction limits
Tracks limit usage
6. Compliance Service

KYC verification
Regulatory compliance checks
Transaction monitoring
Part 4: Atomic Balance Management
Aadvik: “How do you ensure balance operations are atomic? This is critical for wallet accuracy.”

Sara: “Atomic balance operations are the foundation of wallet system. We use database transactions with distributed locking to ensure ACID guarantees.”

A. Balance Update Flow

B. Atomic balance update (summary)
The sequence diagram above shows a single-wallet debit under one distributed lock. For P2P, lock both wallets in a fixed order (e.g. lexicographic walletId) to avoid deadlock, then run one database transaction: SELECT … FOR UPDATE on both rows, validate the sender balance, update both balances, insert matching ledger rows, commit, and release locks. After commit, update or invalidate Redis keys such as balance:{walletId}; never use cached balance alone to authorize debits.

C. Distributed Locking Strategy
Why Distributed Locking?

Multiple service instances can process transactions simultaneously
Need to prevent concurrent balance updates on same wallet
Ensure only one transaction modifies balance at a time
Lock Implementation:

Redis-based distributed locks
Lock key: wallet_lock:{walletId}
Lock timeout: 5 seconds (auto-release if process crashes)
Lock acquisition timeout: 100ms (fail fast if lock unavailable)
Lock acquisition and release (Redis):

Acquire: set wallet_lock:{walletId} to a unique token with SET … NX EX (or equivalent) and ~5s TTL; retry with short backoff until the acquisition timeout.
Release: delete the key only if its value still equals your token (compare-and-delete, often via a small Lua script) so a delayed process cannot drop another holder’s lock.
D. Balance Caching Strategy
Why cache balance in Redis?

Reduce database load
Improve latency (Redis hit: ~low ms vs DB round-trip)
Support high read volume for balance display
Redis usage:

Prefer write-through or delete-after-commit for balance:{walletId} so Redis never leads the database after a successful movement
Keys updated or removed immediately after the DB transaction commits
Typical flow:

Balance updated in the database (inside the transaction)
Transaction commits
Set or delete balance:{walletId} in Redis to match the committed state
Read-only balance views may read from Redis; money movement still reads the DB under row lock
Critical rule:

Never authorize debits from Redis alone — use SELECT … FOR UPDATE (or equivalent) on the wallet row
Redis is for non-authoritative balance display and ancillary keys (limits, idempotency, locks)
Part 5: Top-Up Mechanisms
Aadvik: “How do users add money to their wallet? What top-up methods do you support?”

Sara: “We support multiple top-up methods: bank transfer, cards, UPI, and cash deposits. Each method has different processing flows.”

A. Top-Up Flow
Press enter or click to view image in full size

B. Top-Up Methods
1. Bank Transfer (NEFT/IMPS/UPI):

User initiates bank transfer
Money transferred to wallet’s bank account
System detects incoming transfer
Automatically credits wallet
2. Credit/Debit Card:

User enters card details
Payment gateway processes card payment
On success, wallet credited immediately
3. UPI Direct:

User initiates UPI payment
UPI app processes payment
On success, wallet credited immediately
4. Cash Deposit:

User deposits cash at agent location
Agent confirms deposit
Wallet credited after confirmation
C. Top-up processing (summary)
The Top-Up Flow diagram shows the synchronous path through the payment gateway. Operationally: validate the request and per-wallet limits; persist a top-up record (often PENDING). For card/UPI, drive the gateway; on success, credit under wallet lock in one DB transaction with a ledger row, then refresh Redis. For bank transfer, return funding instructions and rely on idempotent callbacks to credit once when money is confirmed.

Part 6: P2P Transfers & Merchant Payments
Aadvik: “How do wallet-to-wallet transfers work? How do you ensure atomicity?”

Sara: “P2P transfers are atomic operations that debit one wallet and credit another in a single transaction.”

A. P2P Transfer Flow
Press enter or click to view image in full size

B. P2P transfer (summary)
The P2P Transfer Flow diagram matches the implementation: enforce limits and eligibility, acquire both Redis wallet locks in deterministic order, then one DB transaction with pessimistic row locks, dual balance updates, paired ledger entries, commit, lock release, and Redis balance key maintenance. Notify the payee asynchronously after success.

C. Merchant Payment Flow
Merchant Payment:

Similar to P2P transfer
Debit user wallet
Credit merchant wallet
Additional merchant fee deduction
Settlement to merchant bank account (T+1)
Merchant payment (summary):

Same atomic pattern as P2P: debit buyer wallet, credit merchant wallet, with ledger entries linking the payment id.
Apply platform or MDR fee per pricing rules (either net settlement in one transaction or a follow-up ledger entry — keep rules consistent with accounting).
Bank settlement (T+n) is a separate payout flow from the merchant wallet or settlement account; the in-wallet leg stays ACID as above.
Part 7: Wallet-to-Bank Transfers
Aadvik: “How do users withdraw money from wallet to bank account?”

Sara: “Wallet-to-bank transfers use bank integration to transfer funds from wallet to user’s bank account.”

A. Withdrawal Flow
Press enter or click to view image in full size

B. Withdrawal processing (summary)
The Withdrawal Flow diagram shows debit then bank rail. In practice: validate limits and beneficiary; create a withdrawal record; debit the wallet under lock in one transaction with a WITHDRAWAL ledger row; call the bank adapter; on bank failure or timeout, reverse with a compensating credit and ledger entry, or drive state via reconciliation jobs. Keep Redis in sync after each successful balance change.

Part 8: Database Design
Aadvik: “What data do you need to store? How do you design the database schema?”

Sara: “We need to store wallet data, transaction ledger, top-up records, and more. Here’s the schema:”

A. Database Schema
1. Wallets Table:

CREATE TABLE wallets (
    wallet_id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL UNIQUE,
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    currency VARCHAR(3) NOT NULL DEFAULT 'INR',
    status VARCHAR(20) NOT NULL,  -- ACTIVE, SUSPENDED, CLOSED
    kyc_status VARCHAR(20) NOT NULL,  -- PENDING, VERIFIED, REJECTED
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
);
2. Transaction Ledger Table:

CREATE TABLE transaction_ledger (
    ledger_id VARCHAR(50) PRIMARY KEY,
    wallet_id VARCHAR(50) NOT NULL,
    transaction_id VARCHAR(50) NOT NULL,
    transaction_type VARCHAR(20) NOT NULL,  -- TOP_UP, P2P_TRANSFER, MERCHANT_PAYMENT, WITHDRAWAL
    amount DECIMAL(15, 2) NOT NULL,
    balance_before DECIMAL(15, 2) NOT NULL,
    balance_after DECIMAL(15, 2) NOT NULL,
    direction VARCHAR(10) NOT NULL,  -- CREDIT, DEBIT
    reference_id VARCHAR(100),  -- Related transaction or external reference
    status VARCHAR(20) NOT NULL,  -- SUCCESS, FAILED, PENDING
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (wallet_id) REFERENCES wallets(wallet_id),
    INDEX idx_wallet_id (wallet_id),
    INDEX idx_transaction_id (transaction_id),
    INDEX idx_created_at (created_at)
) PARTITION BY RANGE (created_at);
3. Top-Up Transactions Table:

CREATE TABLE top_up_transactions (
    top_up_id VARCHAR(50) PRIMARY KEY,
    wallet_id VARCHAR(50) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    method VARCHAR(20) NOT NULL,  -- CARD, UPI, BANK_TRANSFER, CASH
    payment_provider_id VARCHAR(50),
    payment_transaction_id VARCHAR(100),
    status VARCHAR(20) NOT NULL,  -- PENDING, SUCCESS, FAILED
    created_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    FOREIGN KEY (wallet_id) REFERENCES wallets(wallet_id),
    INDEX idx_wallet_id (wallet_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);
4. P2P Transfers Table:

CREATE TABLE p2p_transfers (
    transfer_id VARCHAR(50) PRIMARY KEY,
    from_wallet_id VARCHAR(50) NOT NULL,
    to_wallet_id VARCHAR(50) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    status VARCHAR(20) NOT NULL,  -- SUCCESS, FAILED, PENDING
    created_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    FOREIGN KEY (from_wallet_id) REFERENCES wallets(wallet_id),
    FOREIGN KEY (to_wallet_id) REFERENCES wallets(wallet_id),
    INDEX idx_from_wallet_id (from_wallet_id),
    INDEX idx_to_wallet_id (to_wallet_id),
    INDEX idx_status (status)
);
5. Withdrawals Table:

CREATE TABLE withdrawals (
    withdrawal_id VARCHAR(50) PRIMARY KEY,
    wallet_id VARCHAR(50) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    bank_account_number VARCHAR(50) NOT NULL,
    bank_ifsc VARCHAR(20) NOT NULL,
    bank_transfer_id VARCHAR(100),
    status VARCHAR(20) NOT NULL,  -- PENDING, SUCCESS, FAILED
    created_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    FOREIGN KEY (wallet_id) REFERENCES wallets(wallet_id),
    INDEX idx_wallet_id (wallet_id),
    INDEX idx_status (status)
);
6. Wallet Limits Table:

CREATE TABLE wallet_limits (
    limit_id VARCHAR(50) PRIMARY KEY,
    wallet_id VARCHAR(50) NOT NULL,
    limit_type VARCHAR(20) NOT NULL,  -- BALANCE_LIMIT, DAILY_TRANSFER_LIMIT, MONTHLY_TRANSFER_LIMIT
    limit_value DECIMAL(15, 2) NOT NULL,
    current_usage DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    reset_date DATE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    FOREIGN KEY (wallet_id) REFERENCES wallets(wallet_id),
    UNIQUE KEY uk_wallet_limit_type (wallet_id, limit_type),
    INDEX idx_wallet_id (wallet_id)
);
B. Database Sharding
Sharding Strategy:

Shard wallets table by wallet_id (hash-based)
Shard transaction ledger by wallet_id (same shard as wallet)
Each shard handles subset of wallets
Enables horizontal scaling
Partitioning:

Partition transaction ledger by date (monthly partitions)
Archive old partitions
Improves query performance
Part 9: API Design
Aadvik: “What APIs do you expose? How do you design the API endpoints?”

Sara: “We expose APIs for wallet operations, top-up, transfers, and more. Here are the key endpoints:”

A. Wallet APIs
Get Wallet Balance
Endpoint: GET /api/v1/wallets/{wallet_id}/balance

Response (200 OK):

{
  "wallet_id": "wallet_123",
  "balance": 10000.00,
  "currency": "INR",
  "updated_at": "2024-01-15T10:30:00Z"
}
Get Transaction History
Endpoint: GET /api/v1/wallets/{wallet_id}/transactions?page=1&limit=20

Response (200 OK):

{
  "transactions": [
    {
      "transaction_id": "txn_789",
      "type": "P2P_TRANSFER",
      "amount": 1000.00,
      "direction": "DEBIT",
      "balance_after": 9000.00,
      "created_at": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1000
  }
}
B. Top-Up APIs
Initiate Top-Up
Endpoint: POST /api/v1/wallets/{wallet_id}/top-up

Request:

{
  "amount": 5000.00,
  "method": "UPI",
  "payment_details": {
    "upi_id": "user@paytm"
  }
}
Response (200 OK):

{
  "top_up_id": "topup_456",
  "amount": 5000.00,
  "method": "UPI",
  "status": "PENDING",
  "payment_url": "https://payment.com/pay/topup_456",
  "created_at": "2024-01-15T10:30:00Z"
}
C. Transfer APIs
P2P Transfer
Endpoint: POST /api/v1/wallets/{wallet_id}/transfer

Request:

{
  "to_wallet_id": "wallet_789",
  "amount": 1000.00,
  "remarks": "Dinner payment"
}
Response (200 OK):

{
  "transfer_id": "transfer_123",
  "from_wallet_id": "wallet_123",
  "to_wallet_id": "wallet_789",
  "amount": 1000.00,
  "status": "SUCCESS",
  "created_at": "2024-01-15T10:30:00Z"
}
Pay Merchant
Endpoint: POST /api/v1/wallets/{wallet_id}/pay-merchant

Request:

{
  "merchant_id": "merchant_456",
  "amount": 2000.00,
  "order_id": "order_789"
}
Response (200 OK):

{
  "payment_id": "payment_123",
  "merchant_id": "merchant_456",
  "amount": 2000.00,
  "fee": 20.00,
  "status": "SUCCESS",
  "created_at": "2024-01-15T10:30:00Z"
}
D. Withdrawal APIs
Initiate Withdrawal
Endpoint: POST /api/v1/wallets/{wallet_id}/withdraw

Request:

{
  "amount": 5000.00,
  "bank_account": {
    "account_number": "1234567890",
    "ifsc": "HDFC0001234"
  }
}
Response (200 OK):

{
  "withdrawal_id": "withdrawal_123",
  "amount": 5000.00,
  "status": "PROCESSING",
  "estimated_completion": "2024-01-16T10:30:00Z",
  "created_at": "2024-01-15T10:30:00Z"
}
Part 10: Failure Handling & Resilience
Aadvik: “What happens when things go wrong? How do you handle failures?”

Sara: “We handle failures gracefully with transaction rollback, idempotency, and retry mechanisms.”

A. Failure Scenarios
1. Insufficient Balance:

Problem: User tries to transfer more than wallet balance

Mitigation:

Check balance before transaction
Fail fast with clear error message
Recovery: User can add money and retry

2. Lock Acquisition Failure:

Problem: Cannot acquire distributed lock (high contention)

Mitigation:

Retry lock acquisition (exponential backoff)
Maximum 3 retries
Fail with clear error message
Recovery: User can retry transaction

3. Database Transaction Failure:

Problem: Database transaction fails during balance update

Mitigation:

Automatic rollback
Release distributed lock
Return error to user
Recovery: Transaction is rolled back, user can retry

4. Top-Up Payment Failure:

Problem: Payment provider fails during top-up

Mitigation:

Top-up marked as failed
No wallet credit
User can retry top-up
Recovery: User can retry with different payment method

5. Withdrawal Bank Transfer Failure:

Problem: Bank transfer fails after wallet debit

Mitigation:

Reverse wallet debit (credit back)
Mark withdrawal as failed
Alert operations team
Recovery: Manual intervention or automatic retry

B. Idempotency
Why Idempotency is Critical:

Network retries can cause duplicate requests
User might retry transaction
System must handle duplicate requests gracefully
Implementation:

Each transaction request has unique transaction_id
Check if transaction_id already exists
If exists, return existing transaction status (don’t process again)
If new, process transaction
Idempotency Key:

Format: wallet_id + transaction_type + timestamp + random
Stored in Redis with transaction response
TTL: 24 hours
Part 11: Scaling Strategies
Aadvik: “How do you scale this system to handle 15,000+ TPS?”

Sara: “Scaling requires multiple strategies across different components:”

A. Horizontal Scaling
1. Wallet Service:

Stateless services → scale horizontally
Load balancer distributes requests
Auto-scale based on TPS and latency
Each instance handles subset of wallets
2. Transfer Service:

Stateless transfer processing
Scale horizontally
Lock contention handled by distributed locks
3. Top-Up Service:

Stateless top-up processing
Scale horizontally
Payment gateway handles payment processing
B. Database Scaling
1. Read Replicas:

Read replicas for read-heavy workloads
Balance inquiries use read replicas
Separate read and write paths
2. Sharding:

Shard wallets table by wallet_id (hash-based)
Each shard handles subset of wallets
Enables horizontal scaling
3. Partitioning:

Partition transaction ledger by date (monthly partitions)
Archive old partitions
Improves query performance
C. Caching Strategy
Redis (distributed cache) and database:

Redis

Balance reads for non-authoritative display (write-through or invalidate-on-write after DB commit; never sole source for debits)
Wallet limits (e.g. 10 min TTL)
Transaction status (e.g. 1 hour TTL)
Idempotency keys (e.g. 24 hour TTL)
Distributed locks (~5 second TTL)
Database

Source of truth for balances and ledger
All writes and authoritative reads for money movement
D. Distributed Locking Scaling
1. Redis Cluster:

Multiple Redis instances for lock storage
Distributed locks work across cluster
High availability
2. Lock Sharding:

Locks distributed across Redis cluster
Reduces lock contention
Improves throughput
Part 12: Monitoring & Observability
Aadvik: “How do you monitor this system?”

Sara: “Comprehensive monitoring is essential for a wallet system, especially for balance accuracy.”

A. Key Metrics
1. Transaction Metrics:

Transaction success rate
Transaction latency (p50, p95, p99)
Transactions per second (TPS)
Failed transactions count
Transfer success rate
2. Balance Metrics:

Balance accuracy (reconciliation)
Balance update latency
Lock acquisition time
Lock contention rate
3. Business Metrics:

Total wallet balance (sum of all wallets)
Active wallets
Average wallet balance
Top-up volume (daily, monthly)
Transfer volume (daily, monthly)
Withdrawal volume (daily, monthly)
4. System Metrics:

API response time
Database query latency
Redis cache hit rate
Lock acquisition failures
Error rates
5. Compliance Metrics:

KYC completion rate
Limit violations
Suspicious transactions
Compliance alerts
B. Alerting Rules
Critical Alerts:

Balance discrepancy detected
Transaction success rate < 99%
Lock acquisition failure rate > 1%
Database connection pool > 90%
Error rate > 0.1%
Warning Alerts:

Transaction latency p95 > 500ms
Cache hit rate < 80%
Lock contention > 5%
Balance update latency > 100ms
C. SLOs & SLIs
Service Level Indicators:

Transaction success rate: 99.9%
Transaction latency: p95 < 500ms
Balance check latency: p95 < 100ms
API availability: 99.99%
Zero balance discrepancies
Service Level Objectives:

99.9% of transactions complete successfully
95% of transactions complete within 500ms
99.99% API uptime
Zero balance loss or duplication
Error Budget:

0.1% error budget per month
If exceeded: stop feature work, focus on reliability
Part 13: Cost Analysis
Aadvik: “What’s the infrastructure cost for running a digital wallet system at this scale?”

Sara: “Let me break down the infrastructure costs. I’ll use AWS pricing as reference.”

⚠️ Important Cost Disclaimer
Critical Note: The cost estimates provided below are rough approximations and assumptions based on publicly available pricing information and typical cloud provider rates. These numbers are NOT actual billing amounts and should be treated as illustrative estimates only.

A. Infrastructure Components
1. Compute (EC2/ECS):

Wallet services: 25 instances (c5.2xlarge) = ~$3,750/month
Transfer services: 15 instances (c5.xlarge) = ~$2,250/month
Top-up services: 10 instances (c5.xlarge) = ~$1,500/month
Total Compute: ~$7,500/month
2. Database (RDS PostgreSQL):

Primary: db.r5.4xlarge (multi-AZ) = ~$2,000/month
Read replicas: 5 × db.r5.2xlarge = ~$5,000/month
Total Database: ~$7,000/month
3. Caching (ElastiCache Redis):

5 × cache.r5.xlarge = ~$2,500/month
Total Cache: ~$2,500/month
4. Messaging (Kafka/MSK):

5 × kafka.m5.large = ~$800/month
Total Messaging: ~$800/month
5. Network & Data Transfer:

Data transfer: ~$1,500/month
Total Network: ~$1,500/month
6. Storage:

Transaction data: 42TB × 0.023/GB= 0.023/GB= 1,000/month
Total Storage: ~$1,000/month
Total Estimated Monthly Cost: ~$20,300/month

Note: Actual costs vary significantly based on:

Reserved instances (30–70% discount)
Enterprise discounts
Actual usage patterns
Regional pricing differences
Optimization strategies
Part 14: Trade-offs & Future Improvements
Aadvik: “What trade-offs did you make? What would you improve?”

Sara: “Important trade-offs:”

1. Distributed Locking: Performance vs Consistency

Chosen: Distributed locks for balance operations
Trade-off: Additional latency for lock acquisition
Mitigation: Fast lock acquisition (< 100ms), lock timeout
Benefit: Guaranteed balance consistency
2. Balance Caching: Accuracy vs Performance

Chosen: Write-through cache pattern
Trade-off: Cache updated after transaction (slight delay)
Mitigation: Never use cache for debit operations, always read from DB
Benefit: Balance accuracy guaranteed
3. Database Transactions: Consistency vs Performance

Chosen: ACID transactions for balance updates
Trade-off: Database lock contention
Mitigation: Sharding, read replicas, connection pooling
Benefit: Zero balance discrepancies
4. Top-Up Processing: Real-Time vs Batch

Chosen: Real-time top-up processing
Trade-off: Higher compute costs
Mitigation: Async processing where possible
Benefit: Better user experience
Aadvik: “What would you improve if you had more time?”

Sara: “Additional improvements:

Event Sourcing: Use event sourcing for transaction ledger (better audit trail)
CQRS Pattern: Separate read and write models for better performance
Saga Pattern: For complex multi-step operations (top-up + transfer)
Advanced Fraud Detection: ML-based fraud detection for suspicious transactions
Multi-Region Deployment: Deploy in multiple regions for resilience
Predictive Scaling: ML-based prediction of transaction volume
Advanced Analytics: Real-time wallet analytics dashboard
Blockchain Integration: For transparent transaction history (optional)”
Summary
Key Takeaways:

Atomic Balance Operations — ACID transactions with distributed locking ensure balance accuracy
Distributed Locking — Prevents race conditions in concurrent balance updates
Top-Up Mechanisms — Multiple payment methods for adding money to wallet
P2P Transfers — Atomic wallet-to-wallet transfers with transaction ledger
Merchant Payments — Wallet payments to merchants with fee calculation
Wallet-to-Bank Transfers — Withdrawal functionality with bank integration
System Handles:

500 million transactions per month
15,000+ TPS (peak)
100 million wallet users
1 million merchants
Sub-500ms transfer operations
99.99% availability
Zero balance discrepancies
Architecture Highlights:

ACID transactions for balance accuracy
Distributed locking for consistency
Write-through caching for performance
Transaction ledger for complete audit trail
Comprehensive monitoring with SLOs
This architecture powers digital wallet systems at massive scale with guaranteed balance accuracy and atomic operations!


Link : https://codefarm0.medium.com/designing-a-digital-wallet-system-balance-management-top-up-p2p-transfers-eed46f757dc7