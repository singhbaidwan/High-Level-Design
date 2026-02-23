# Active-Active vs Active-Passive

Source: extracted from `architects/interview-ready.md`

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

![Consolidated C](../images/summary-31.png)

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

