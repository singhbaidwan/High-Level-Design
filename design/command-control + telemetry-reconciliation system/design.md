
This is a command-control + telemetry-reconciliation system, so the central design principle is: command intent is immutable, device observations are append-only, and derived state is continuously reconciled.
1. Requirements
Functional requirements
1. Create a command campaign, for example:
Reduce AC power consumption by 20%
Region: Punjab
Start: 14:00
Deadline: 14:15
1. Resolve the target population and create an immutable target snapshot.
2. Dispatch commands to millions of intermittently connected devices.
3. Track separate lifecycle states:
TARGETED
DELIVERY_ATTEMPTED
DELIVERED
ACCEPTED
REJECTED
EXECUTED
MEASURED
EXPIRED
UNKNOWN
1. Retry delivery for offline/unresponsive devices with bounded retries.
2. Guarantee device-side idempotency.
3. Ensure an old delayed command never overrides a newer command.
4. Ingest authenticated telemetry asynchronously.
5. Correlate every ACK, execution report, and measurement to:
device_id
campaign_id
command_sequence/config_version
event_time
1.  Continuously calculate campaign progress.
2.  Support late-arriving telemetry and revise aggregates.
3.  Run reconciliation for devices whose state remains uncertain.
4.  Feed reliable campaign outcomes into future planning.
5.  Provide operator safety controls, audit trails, observability, and emergency cancellation.
## 2. Non-functional requirements

Assume an illustrative scale:

```text
Registered devices             50M
Devices simultaneously online  10M–20M
Campaigns/day                   1,000
Largest campaign               10M devices
Command dispatch peak          500K–1M devices/sec
Telemetry steady state         200K events/sec
Telemetry burst                1M+ events/sec
Campaign deadline              5–30 minutes
Telemetry lateness             seconds → hours
Raw telemetry retention        30–90 days
Audit retention                multiple years
```

The important properties are:

- **High availability:** control plane should survive regional/component failures.
- **Durability:** once a campaign is accepted, its target set and command must never disappear.
- **At-least-once delivery:** duplicate delivery is acceptable because devices are idempotent.
- **No stale overwrite:** newer control versions fence older commands.
- **Eventual correctness:** campaign progress may change when late telemetry arrives.
- **Near-real-time visibility:** operators should see progress within seconds.
- **Massive fan-out:** sending one campaign must not produce 10M synchronous API calls from one service.
- **Backpressure:** a campaign must not overwhelm brokers, devices, or a regional network.
- **Security:** authenticated devices, signed commands, encrypted communication.
- **Auditability:** every campaign, approval, dispatch, response, reconciliation, and override is recorded.
- **Safety over availability:** failure to determine device state must never trigger unsafe automatic compensation.

---

# 3. Core architecture

```text
                         CONTROL PLANE
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│ Operator / Automated Planner                                         │
│             │                                                        │
│             ▼                                                        │
│     ┌─────────────────┐                                              │
│     │ Campaign API    │                                              │
│     └────────┬────────┘                                              │
│              │                                                       │
│       ┌──────▼────────┐      ┌─────────────────┐                     │
│       │ Safety/Policy │─────►│ Approval/Audit  │                     │
│       │ Engine        │      └─────────────────┘                     │
│       └──────┬────────┘                                              │
│              │                                                       │
│       ┌──────▼──────────┐                                            │
│       │ Target Resolver │──────► Device Inventory                    │
│       └──────┬──────────┘                                            │
│              │                                                       │
│       immutable snapshot                                             │
│              │                                                       │
│              ▼                                                       │
│      Object Store / Snapshot DB                                      │
│              │                                                       │
│              ▼                                                       │
│      ┌─────────────────┐                                             │
│      │ Fanout Planner  │                                             │
│      └────────┬────────┘                                             │
└───────────────┼──────────────────────────────────────────────────────┘
                │
                ▼
          Command Kafka
                │
       partition by region/shard
                │
      ┌─────────┴─────────┐
      ▼                   ▼
 Regional Dispatcher   Regional Dispatcher
      │                   │
      ▼                   ▼
 MQTT / Device Gateway / Long-lived connections
      │
      ▼
   DEVICES
      │
      │ ACK / ACCEPT / REJECT / EXECUTION / TELEMETRY
      ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         DATA PLANE                                   │
│                                                                      │
│ Device Gateway                                                       │
│      │                                                               │
│      ▼                                                               │
│ Telemetry Kafka                                                      │
│      │                                                               │
│ ┌────┴─────────┬─────────────┬─────────────────┐                     │
│ ▼              ▼             ▼                 ▼                     │
│ Dedupe      Device State   Event-time       Raw Event                │
│ Processor   Processor      Aggregator       Archive                  │
│               │               │                 │                    │
│               ▼               ▼                 ▼                    │
│        Device State DB   Progress Store     Object Store             │
│                               │                                      │
│                               ▼                                      │
│                      Campaign Dashboard                              │
└──────────────────────────────────────────────────────────────────────┘

                     RECONCILIATION
                              │
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
     Unknown-state scanner             Telemetry reprocessing
            │                                   │
            ▼                                   ▼
      targeted retries                    corrected state
            │                                   │
            └──────────────► aggregates ◄────────┘
```

The architecture deliberately separates:

```text
Command intent
      ↓
Delivery
      ↓
Device decision
      ↓
Execution
      ↓
Observed physical effect
```

A common design mistake would be treating these as one `SUCCESS/FAILED` field.

---

# 4. APIs

### Create campaign

```http
POST /v1/campaigns
```

```json
{
  "commandType": "REDUCE_POWER",
  "parameters": {
    "percentage": 20
  },
  "target": {
    "region": "punjab",
    "deviceType": "AIR_CONDITIONER"
  },
  "startTime": "2026-09-07T14:00:00Z",
  "deadline": "2026-09-07T14:15:00Z",
  "maxRetries": 5
}
```

Response:

```json
{
  "campaignId": "cmp-87932",
  "status": "TARGET_RESOLUTION_PENDING"
}
```

---

### Campaign metadata

```http
GET /v1/campaigns/{campaignId}
```

---

### Campaign progress

```http
GET /v1/campaigns/{campaignId}/progress
```

Example:

```json
{
  "targeted": 10000000,
  "deliveryAttempted": 9200000,
  "delivered": 8300000,
  "accepted": 7800000,
  "rejected": 500000,
  "executed": 7500000,
  "measured": 6800000,
  "unknown": 1200000,

  "measuredReductionMW": 742,
  "measurementCoverage": 0.68,

  "aggregateRevision": 17,
  "watermark": "2026-09-07T14:23:00Z"
}
```

---

### Inspect individual device

```http
GET /v1/campaigns/{campaignId}/devices/{deviceId}
```

Useful for debugging:

```text
campaign
   ↓
dispatch attempt
   ↓
delivery ACK
   ↓
acceptance
   ↓
execution
   ↓
measurements
```

---

### Stop further dispatch

Because the **campaign definition is immutable**, I would not modify the campaign command itself.

Instead:

```http
POST /v1/campaigns/{campaignId}/abort
```

creates an immutable administrative action:

```text
CAMPAIGN_ABORT_REQUESTED
```

The dispatcher stops sending unsent work.

For devices that already applied the configuration, we may need a **new superseding campaign** to restore/change their configuration.

This preserves the audit history.

---

# 5. Core entities

## Campaign

```text
Campaign
--------
campaign_id
command_type
command_payload
command_namespace
control_version
created_at
start_at
deadline
retry_policy_id
target_snapshot_id
target_snapshot_hash
created_by
approval_policy
```

Immutable.

---

## TargetSnapshot

```text
TargetSnapshot
--------------
snapshot_id
campaign_id
created_at
device_count
object_store_uri
snapshot_hash
partition_count
```

For 10M targets, I would not store the entire target list inside PostgreSQL.

Use something like:

```text
S3/GCS

campaigns/
   cmp-87932/
      targets/
         region=punjab/
             shard=0001.parquet
             shard=0002.parquet
             ...
```

Each row could contain:

```text
device_id
delivery_region
gateway_id
device_type
control_group
```

The snapshot is critical.

Suppose:

```text
14:00 campaign created
Target = all ACs in Punjab

14:03 device moves to Haryana
```

That must **not retroactively change campaign membership**.

We measure against the original snapshot.

---

# 6. Why snapshot targeting instead of querying devices during dispatch?

Imagine the dispatcher repeatedly does:

```sql
SELECT *
FROM devices
WHERE region = 'Punjab';
```

while sending commands.

The dataset could change during a 10-minute dispatch.

You could end up with:

```text
device A counted twice
device B missed
device C unexpectedly added
```

and later have no deterministic denominator for campaign progress.

Instead:

```text
Device Inventory
       │
       ▼
 Target query
       │
       ▼
 immutable result
       │
       ▼
Target Snapshot
       │
       ▼
Fanout
```

Now:

```text
campaign denominator = snapshot.size
```

forever.

---

# 7. Device-command state

I would **not** model this as merely:

```text
PENDING
SUCCESS
FAILED
```

I would actually store different dimensions.

```text
DeviceCampaignState
-------------------

device_id
campaign_id

delivery_state
    NOT_ATTEMPTED
    ATTEMPTED
    DELIVERED
    DELIVERY_EXPIRED

decision_state
    UNKNOWN
    ACCEPTED
    REJECTED

execution_state
    UNKNOWN
    STARTED
    EXECUTED
    FAILED

measurement_state
    NONE
    OBSERVED
    VERIFIED

latest_control_version
latest_event_time
last_ingestion_time

attempt_count
last_attempt_time

reject_reason
failure_reason

state_revision
```

Why?

Consider:

```text
DELIVERED
+
REJECTED
```

versus:

```text
DELIVERY_EXPIRED
+
UNKNOWN
```

These mean completely different things.

---

# 8. Device state progression

Typical successful path:

```text
TARGETED
   │
   ▼
DELIVERY ATTEMPT
   │
   ▼
DELIVERED ACK
   │
   ▼
ACCEPTED
   │
   ▼
EXECUTION STARTED
   │
   ▼
EXECUTED
   │
   ▼
TELEMETRY OBSERVED
   │
   ▼
MEASURED EFFECT VERIFIED
```

Rejected:

```text
TARGETED
   ↓
DELIVERED
   ↓
REJECTED

Reason:
- local safety constraint
- unsupported command
- temperature already below minimum
- device malfunction
- user override
```

Offline:

```text
TARGETED
   ↓
ATTEMPTED
   ↓
no delivery ACK
   ↓
retry
   ↓
retry
   ↓
deadline
   ↓
DELIVERY_EXPIRED / UNKNOWN
```

These states answer follow-up question #1 immediately.

### Rejected

We know:

```text
DELIVERED = true
decision = REJECTED
```

### Never received

We know only:

```text
attempted = true
delivery acknowledgment = missing
```

Therefore:

```text
delivery = UNKNOWN / EXPIRED
decision = UNKNOWN
```

Never convert missing information into rejection.

---

# 9. Fan-out design

Suppose a campaign targets:

```text
10 million devices
```

The Campaign Service absolutely should not perform:

```java
for (Device device : devices) {
    httpClient.post(device);
}
```

Instead:

```text
Campaign
   │
   ▼
Target Snapshot
   │
   ▼
Fanout Planner
   │
   ├── Region 1 / shard 001
   ├── Region 1 / shard 002
   ├── Region 2 / shard 001
   └── ...
           │
           ▼
     Durable command stream
```

For example:

```text
Kafka topic: command-dispatch

partition key:
    region + delivery_shard
```

Message:

```json
{
  "campaignId": "cmp-87932",
  "snapshotPartition": 417,
  "startOffset": 0,
  "endOffset": 4999
}
```

Notice that this need not be **one Kafka message per device** initially.

We can use:

```text
Campaign
  ↓
Target partition work item
  ↓
Dispatcher reads 5,000 device IDs
  ↓
rate-limited device delivery
```

This keeps control-plane queue volume manageable.

---

# 10. Why Kafka?

Kafka provides:

```text
durable dispatch work
partition ordering
consumer-group scaling
replay
backpressure boundary
regional partitioning
```

But Kafka alone **does not tell us whether the device executed a command**.

Its offset only tells us:

> The dispatcher processed this Kafka record.

It says nothing about:

```text
device received?
device rejected?
device applied?
device later rolled back?
measured electricity changed?
```

Therefore Kafka offset cannot serve as device-command state.

---

# 11. Device delivery protocol

Depending on device capabilities:

```text
MQTT
WebSocket
long polling
cellular IoT protocol
vendor-specific gateway
```

MQTT is a natural choice.

```text
Dispatcher
    ↓
MQTT Broker
    ↓
Device
```

Topic:

```text
devices/{deviceId}/commands
```

Payload:

```json
{
  "campaignId": "cmp-87932",
  "commandId": "cmd-d9843",
  "namespace": "POWER_CONTROL",
  "controlVersion": 928734,
  "issuedAt": "...",
  "expiresAt": "...",
  "payload": {
    "reducePowerPercent": 20
  },
  "signature": "..."
}
```

---

# 12. Critical part: idempotency

We deliberately choose:

```text
AT-LEAST-ONCE delivery
+
DEVICE-SIDE IDEMPOTENCY
```

instead of attempting magical exactly-once network delivery.

The device keeps durable state:

```text
lastSeenCommandIds
latestAppliedVersion[namespace]
```

Basic processing:

```java
void process(Command command) {

    if (alreadyProcessed(command.commandId())) {
        resendPreviousResult(command.commandId());
        return;
    }

    if (command.controlVersion() < latestAppliedVersion(command.namespace())) {
        rejectAsStale(command);
        return;
    }

    validate(command);

    Result result = apply(command);

    persistResult(command.commandId(), result);
    persistLatestVersion(
        command.namespace(),
        command.controlVersion()
    );

    sendExecutionReport(result);
}
```

A retry therefore becomes safe.

---

# 13. Version fencing — most important safety property

The scenario explicitly says:

> A later command must not be overwritten by a delayed earlier one.

Imagine:

```text
Campaign A
version = 100
reduce AC 20%

Campaign B
version = 101
restore AC to normal
```

Device receives:

```text
B first
then delayed A
```

Without fencing:

```text
B applied
   ↓
late A applied
   ↓
WRONG STATE
```

Instead device persists:

```text
POWER_CONTROL latestVersion = 101
```

Then A arrives:

```text
incomingVersion = 100
storedVersion   = 101

100 < 101
     ↓
STALE_COMMAND
     ↓
reject
```

Correct result:

```text
Version 101 remains applied
```

---

# 14. Version namespace

Versions should not necessarily be global across every command the device understands.

For example:

```text
THERMOSTAT_CONFIGURATION
POWER_LIMIT
FIRMWARE
SCHEDULE
```

Each can have its own fencing sequence.

Device:

```text
latestVersion:

POWER_LIMIT           → 91832
THERMOSTAT_CONFIG     → 783
FIRMWARE              → 42
```

A firmware update shouldn't automatically fence an unrelated thermostat command.

---

# 15. Where do versions come from?

We need an authoritative monotonic allocator.

For example:

```text
Campaign Service
       ↓
Control Version Service
       ↓
atomic increment / sequence
```

Conceptually:

```sql
UPDATE control_namespace
SET next_version = next_version + 1
WHERE namespace = 'POWER_LIMIT'
RETURNING next_version;
```

At enormous scale I can use range allocation, but the important property is:

```text
newer command ⇒ higher version
```

not wall-clock timestamps.

Never use device time for correctness because device clocks are untrusted.

---

# 16. Retries

Retries should be **bounded by campaign deadline**.

Example:

```text
Attempt 1      t=0
Attempt 2      t=5 sec
Attempt 3      t=20 sec
Attempt 4      t=1 min
Attempt 5      t=4 min
```

with jitter.

```text
delay =
min(
    maxDelay,
    exponentialBackoff(attempt)
)
+ randomJitter
```

But retry behavior depends on current state.

### No delivery acknowledgement

Retry aggressively enough to meet campaign SLA.

```text
ATTEMPTED
but no DELIVERED ACK
```

### Delivered but no acceptance response

Can safely redeliver the same `commandId`.

Device idempotency prevents duplicate application.

### Rejected

Normally:

```text
DO NOT retry
```

unless reject reason explicitly indicates transient state.

For example:

```text
TEMPORARILY_BUSY
```

may be retryable.

But:

```text
UNSUPPORTED_COMMAND
SAFETY_CONSTRAINT
USER_OVERRIDE
```

should not be blindly retried.

---

# 17. Retry scheduler

I would avoid millions of individual cron jobs.

Options include:

```text
Database deadline scan
Redis sorted sets
Kafka delay/retry topics
timing wheel
```

For this scale, a practical combination is:

```text
short delays:
    retry topics / timing wheel

long-lived reconciliation:
    persistent state DB scanner
```

Conceptually:

```text
Device state
   │
   │ next_retry_at
   ▼
Retry Scheduler
   │
   ▼
retry-dispatch Kafka
   │
   ▼
Regional Dispatcher
```

Correctness remains in the persistent state store.

Redis can accelerate scheduling, but I would avoid making Redis the sole source of truth.

---

# 18. Telemetry event format

Every observation should contain enough correlation data.

```json
{
  "eventId": "evt-127819",
  "deviceId": "ac-918",
  "campaignId": "cmp-87932",
  "commandId": "cmd-d9843",
  "controlVersion": 928734,

  "type": "EXECUTION_COMPLETED",

  "deviceEventTime": "2026-09-07T14:04:21Z",
  "deviceSequence": 129318,

  "reportedState": {
    "powerLimit": 0.8
  },

  "measurement": {
    "powerWatts": 1520
  },

  "signature": "..."
}
```

Correlate everything back to:

```text
campaign
command
device
version
```

This is what the prompt means by:

> Correlate every observation.

---

# 19. Telemetry ingestion pipeline

```text
Devices
   │
   ▼
Gateway
   │
Authenticate device
   │
   ▼
Telemetry Kafka
   │
   ├───────────────► Raw event archive
   │
   ▼
Validation
   │
   ▼
Deduplication
   │
   ▼
Event-time processing
   │
   ├────────────► per-device state
   │
   └────────────► campaign aggregates
```

A technology choice could be:

```text
Kafka
+
Flink / Beam / Dataflow
```

because we specifically care about:

```text
event time
watermarks
late events
window corrections
stateful processing
deduplication
```

---

# 20. Authentication

Device identity is authenticated even though delivery timing is untrusted.

Potential model:

```text
Device certificate
      │
      ▼
mTLS connection
      │
      ▼
Gateway verifies certificate
      │
      ▼
device_id extracted from certificate identity
```

The backend should not accept:

```json
{
  "deviceId": "whatever-the-client-says"
}
```

as trustworthy identity.

Bind:

```text
authenticated certificate identity
         ==
telemetry.device_id
```

Commands should additionally be signed so compromised transport infrastructure cannot fabricate commands.

---

# 21. Deduplication

Because network retries happen, telemetry may arrive multiple times.

Use:

```text
event_id
```

or preferably a device-generated tuple:

```text
(device_id, device_sequence)
```

Example:

```text
AC-718 sequence 100
AC-718 sequence 101
AC-718 sequence 101  ← duplicate
AC-718 sequence 102
```

Processor keeps:

```text
last processed sequence
```

plus a bounded dedupe structure where out-of-order events are possible.

---

# 22. Why event time instead of processing time?

Suppose device is offline:

```text
14:05 executes command
14:06 loses network
14:37 reconnects
14:37 telemetry reaches backend
```

If we use ingestion time:

```text
execution = 14:37
```

which is false.

Telemetry should record:

```text
event_time  = 14:05
ingest_time = 14:37
```

Both are useful.

---

# 23. But device clocks are untrusted

Correct.

So device event time cannot be blindly trusted.

Ideally include:

```text
device monotonic sequence
server reception time
device wall clock
last gateway sync information
```

Then define bounded trust.

For example:

```text
device clock skew <= ±2 min:
    use corrected device event time

clock suspicious:
    mark timestamp confidence LOW

impossible timestamp:
    reject timestamp but retain event
```

The important ordering primitive for device commands remains:

```text
controlVersion
```

not timestamp.

---

# 24. Watermarks and late telemetry

Suppose our progress window is:

```text
14:00 → 14:15
```

At 14:16 we might have:

```text
7M measured devices
2M unknown
1M rejected/offline
```

But another 800K devices reconnect by 14:45.

Therefore campaign statistics cannot be a permanently finalized integer at 14:15.

Use a watermark.

For example:

```text
normal watermark:
eventTime < now - 5 minutes
```

Once the watermark advances, the system can publish an initial stable aggregate.

But we still accept late events.

---

# 25. Revisioned aggregates

Instead of:

```text
campaign progress = immutable final answer
```

store:

```text
CampaignAggregate
-----------------

campaign_id
revision
calculated_at
watermark

target_count
delivered_count
accepted_count
rejected_count
executed_count
measured_count
unknown_count

measured_power_delta

late_event_count
measurement_coverage
confidence
```

Example:

```text
Revision 10 at 14:20
Executed: 7.2M

Revision 11 at 14:27
Executed: 7.35M

Revision 12 at 14:42
Executed: 7.62M
```

The history is extremely useful for auditing.

---

# 26. Late event correction

Suppose initial state:

```text
Device X
UNKNOWN
```

At 15:30 telemetry arrives showing:

```text
executed at 14:08
```

Processing becomes:

```text
Telemetry
   ↓
dedupe
   ↓
locate campaign/device
   ↓
UNKNOWN → EXECUTED
   ↓
aggregate correction

unknown_count   -1
executed_count  +1
```

This is why campaign aggregation should support **retractions/upserts**, not only increment-only counters.

---

# 27. Do not rely on simple counters alone

A dangerous implementation is:

```java
if (event.type == EXECUTED) {
    redis.incr("campaign:123:executed");
}
```

A duplicate event now increments twice.

And a late correction cannot easily undo an earlier classification.

Instead:

```text
event
   ↓
derive current device state
   ↓
compare old state vs new state
   ↓
emit state delta
```

Example:

```text
old:
UNKNOWN

new:
EXECUTED

delta:
UNKNOWN   -1
EXECUTED  +1
```

This makes aggregation deterministic.

---

# 28. Per-device state store

I would use something like:

```text
Cassandra
DynamoDB
Bigtable
ScyllaDB
```

rather than a single relational table for hundreds of millions/billions of device-campaign rows.

Logical schema:

```text
CampaignDeviceState
-------------------

PK:
    campaign_id
    bucket

SK:
    device_id
```

where:

```text
bucket = hash(device_id) % N
```

Example:

```text
(cmp-87932, bucket-147, device-ABC)
```

Benefits:

```text
parallel campaign scans
distributed updates
no hot campaign partition
```

---

# 29. Secondary device-centric view

Operators may also ask:

> What commands were recently sent to device X?

So maintain another materialized view:

```text
DeviceCommandHistory
--------------------

PK = device_id
SK = command_version/time
```

This can be asynchronously maintained.

Don't force one database index to satisfy both access patterns at massive scale.

---

# 30. Database design

I would divide ownership like this:

| Data | Storage |
|---|---|
| Campaign definitions | PostgreSQL |
| Approvals / safety policies | PostgreSQL |
| Target snapshots | Object storage / Parquet |
| Device inventory | distributed inventory DB/search index |
| Per-device campaign state | Cassandra/Bigtable/DynamoDB |
| Command streams | Kafka |
| Telemetry streams | Kafka |
| Raw telemetry | Object storage |
| Live aggregates | distributed KV/OLAP store |
| Historical analytics | warehouse/lake |
| Audit events | immutable log + object storage |

Notice that Redis is optional.

I would use it for:

```text
rate limiting
short-lived retry scheduling
dashboard cache
```

but not authoritative campaign state.

---

# 31. Reconciliation

Streaming alone isn't enough.

We need a separate reconciliation process.

```text
                 Target Snapshot
                       │
                       ▼
           expected device population
                       │
                       │ compare
                       ▼
              Device State Store
                       │
                       ▼
             Missing / inconsistent
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
    Retry delivery            Query telemetry/
                              recompute state
```

Periodic jobs might scan:

```text
campaigns before deadline
campaigns shortly after deadline
campaigns +1 hour
campaigns +24 hours
```

---

# 32. Reconciliation categories

Classify unresolved devices.

```text
UNKNOWN_NOT_CONTACTED
UNKNOWN_DELIVERY
DELIVERED_NO_DECISION
ACCEPTED_NO_EXECUTION
EXECUTED_NO_MEASUREMENT
MEASUREMENT_INSUFFICIENT
```

Each category has different action.

### Example

```text
DELIVERED_NO_DECISION
```

Potentially resend request/query status.

But:

```text
EXECUTED_NO_MEASUREMENT
```

should not resend the command.

Instead request telemetry.

Otherwise a reconciliation process could accidentally cause repeated physical action.

---

# 33. Device reconnect path

When an offline device reconnects:

```text
Device
  │
  ▼
Gateway authentication
  │
  ▼
Device session created
  │
  ▼
Command eligibility lookup
```

Suppose queued commands:

```text
v100
v101
v102
```

and device already has:

```text
v101
```

The server can optimize by sending only:

```text
v102
```

But correctness must still exist on the device.

Even if the backend accidentally delivers:

```text
v100
```

the device rejects it through version fencing.

That gives defense in depth.

---

# 34. Compliance vs measured effect

The question asks whether compliance is based on explicit execution or observed effect.

A strong design should keep them separate.

For device X:

```text
command accepted      YES
execution reported    YES
expected reduction    20%
observed reduction    12%
```

What happened?

Possibilities include:

```text
measurement noise
compressor cycle
user load changes
device misreporting
command partially effective
incorrect baseline
```

Therefore store both:

```text
execution compliance

and

physical effect
```

---

# 35. Measurement model

Before campaign:

```text
baseline_power(device, t)
```

During campaign:

```text
actual_power(device, t)
```

Effect:

```text
effect =
baselinePower - observedPower
```

Campaign:

```text
Σ effect(device)
```

But we shouldn't naïvely compare one telemetry sample.

Baseline could include:

```text
historic same-time consumption
temperature
device model
previous 15-minute average
regional conditions
```

That's more analytics/modeling than core dispatch, but the architecture should support it.

---

# 36. Progress semantics when many devices are offline

This is one of the most important interview points.

Imagine:

```text
Total targets = 10M

Measured     = 4M
Executed     = 3.8M
Rejected     = 0.2M
Unknown      = 6M
```

Reporting:

```text
"95% execution success"
```

because:

```text
3.8M / 4M = 95%
```

would be extremely misleading.

Instead publish multiple denominators.

### Fleet coverage

```text
measured / targeted
= 4M / 10M
= 40%
```

### Observed execution rate

```text
executed / observed
= 3.8M / 4M
= 95%
```

### Confirmed fleet execution rate

```text
executed / targeted
= 3.8M / 10M
= 38%
```

Dashboard should say something like:

```text
Confirmed fleet execution    38%
Execution among observed     95%
Telemetry coverage           40%
Unknown                      60%
```

Now the operator understands uncertainty.

---

# 37. Confidence

For planning future commands, I would expose something such as:

```text
HIGH confidence
MEDIUM confidence
LOW confidence
```

derived from:

```text
telemetry coverage
device clock quality
measurement availability
reconciliation completeness
late-event rate
sample representativeness
```

Example:

```json
{
  "estimatedReductionMW": 1050,
  "confirmedReductionMW": 742,
  "measurementCoverage": 0.68,
  "confidence": "MEDIUM"
}
```

Never present extrapolated numbers as confirmed measurements.

---

# 38. Later-command planning

Suppose target power reduction was:

```text
1,000 MW
```

Measured:

```text
700 MW
```

Naïve planner:

```text
Shortfall = 300 MW
send another 30% command!
```

That can be dangerous because maybe:

```text
30% of devices simply haven't reported yet.
```

So planner needs:

```text
confirmed measured reduction
estimated reduction
unknown-device fraction
late telemetry profile
regional capacity
safety policy
previous command versions
```

Decision process:

```text
Campaign result
      │
      ▼
Confidence calculation
      │
      ▼
Safety Policy Engine
      │
      ├── enough confidence ──► calculate next action
      │
      └── low confidence ─────► wait / reconcile / human review
```

---

# 39. Safety controls

For infrastructure controlling millions of physical devices, safety should be first-class.

Policy examples:

```text
max reduction per device       20%
max regional reduction         15%
max command ramp               5% / minute
minimum interval between cmds  10 minutes
minimum telemetry coverage     70%
max unknown fraction           20%
```

Potentially:

```text
if unknown_devices > 30%:
    AUTOMATIC FOLLOW-UP DISABLED
```

---

# 40. Human approval

Riskier campaigns can require:

```text
Operator
   ↓
Policy validation
   ↓
Approval 1
   ↓
Approval 2
   ↓
Dispatch
```

For emergency situations:

```text
pre-approved playbook
+
strict boundaries
```

rather than bypassing safety entirely.

---

# 41. Kill switch

Need an operator mechanism for:

```text
STOP FURTHER DISPATCH
```

But remember:

```text
campaign abort ≠ rollback
```

Devices may already have executed the command.

Rollback should normally be another versioned command:

```text
v100 reduce 20%

v101 restore normal
```

This preserves ordering and auditability.

---

# 42. Circuit breakers

Suppose a firmware bug causes:

```text
80% rejection rate
```

in one device model.

Dispatcher should automatically detect:

```text
rejection_rate(model=X) > threshold
```

and pause that segment.

Similarly:

```text
device execution failure spike
broker error spike
gateway latency spike
```

can trigger regional/segment throttling.

---

# 43. Rate limiting

A control campaign targeting 10M devices should not become a DDoS against our own fleet.

Fan-out controller might use:

```text
Global budget:
    500K commands/sec

Region Punjab:
    80K/sec

Gateway X:
    5K/sec

Device model Y:
    20K/sec
```

Hierarchical token buckets work well.

```text
Global limiter
     ↓
Region limiter
     ↓
Gateway limiter
     ↓
dispatch
```

---

# 44. Backpressure

If MQTT brokers begin falling behind:

```text
Kafka backlog ↑
```

That's acceptable.

We would rather let durable Kafka lag increase than drop commands or overload gateways.

Dispatcher should respond:

```text
broker latency ↑
        ↓
consumer dispatch rate ↓
        ↓
Kafka retains work
```

Observability then surfaces:

```text
campaign may miss deadline
```

to operators.

---

# 45. Observability

I would build metrics around the entire funnel.

```text
TARGETED
   ↓
ATTEMPTED
   ↓
DELIVERED
   ↓
ACCEPTED
   ↓
EXECUTED
   ↓
MEASURED
```

For every campaign:

```text
target_count
dispatch_rate
dispatch_lag
delivery_rate
acceptance_rate
rejection_rate
execution_rate
measurement_rate
unknown_rate

p50/p95/p99:
time_to_delivery
time_to_accept
time_to_execute
time_to_measure

retry_rate
retry_success_rate
late_event_rate

telemetry_lag
watermark_lag

measured_effect
expected_effect
```

---

# 46. Segment metrics

Aggregate not only globally but also by:

```text
region
device type
device model
firmware version
network provider
gateway
```

Why?

Suppose overall rejection is:

```text
5%
```

which looks fine.

But:

```text
Firmware 7.2 rejection = 74%
```

Global averages can hide operational disasters.

---

# 47. Tracing one device

Operators need something like:

```text
Campaign cmp-87932
Device ac-10827

14:00:01 TARGETED
14:00:03 DISPATCH_ATTEMPT #1
14:00:03 broker accepted
14:00:05 delivery ACK
14:00:05 ACCEPTED
14:00:07 EXECUTION_STARTED
14:00:12 EXECUTION_COMPLETED
14:01:02 telemetry power=1.5kW
14:02:02 telemetry power=1.4kW
14:03:00 effect VERIFIED
```

That audit trail is invaluable when debugging.

---

# 48. Failure scenarios

## Kafka unavailable

Campaign remains durable.

```text
Campaign DB
Target Snapshot
```

are already committed.

Fan-out resumes once queue availability returns.

---

## Dispatcher crashes

Kafka work isn't committed.

Another consumer receives it.

Potential duplicate send?

Yes.

Device idempotency handles it.

---

## MQTT broker fails

Reconnect clients to another broker.

Outstanding commands are either:

```text
retried
or
redelivered
```

Again idempotency protects devices.

---

## State DB unavailable

Do not acknowledge telemetry processing as complete until durable state/output can be recovered.

Kafka provides replay.

---

## Aggregator crashes

Rebuild aggregates from:

```text
device state changes
or
raw telemetry replay
```

---

## Telemetry arrives twice

Deduplication.

---

## Telemetry arrives hours late

Update device state and publish a new aggregate revision.

---

## Device lies about command version

Identity is trusted, but not necessarily device behavior.

Validate against server-issued campaign metadata:

```text
(device_id, campaign_id, version)
```

If no matching command existed:

```text
INVALID_TELEMETRY
```

audit it rather than contaminating aggregates.

---

# 49. Campaign creation transaction

One subtle design issue:

We must avoid:

```text
campaign exists
but target snapshot doesn't

or

target snapshot exists
but campaign never becomes dispatchable
```

I would use a campaign lifecycle.

```text
DRAFT
  ↓
TARGET_RESOLVING
  ↓
TARGET_FROZEN
  ↓
APPROVED
  ↓
SCHEDULED
  ↓
DISPATCHING
  ↓
DEADLINE_REACHED
  ↓
RECONCILING
  ↓
CLOSED
```

Definition stays immutable once `TARGET_FROZEN`.

Lifecycle state itself lives in separate operational metadata.

---

# 50. Outbox pattern

When campaign becomes dispatchable:

```text
DB transaction:

update CampaignOperationalState = SCHEDULED

insert OutboxEvent(
    CAMPAIGN_READY
)
```

Outbox publisher:

```text
PostgreSQL
   ↓
Outbox
   ↓
Kafka
```

This avoids:

```text
DB commit succeeded
Kafka publish failed
```

leaving campaigns permanently stuck.

---

# 51. Complete write flow

```text
1. Operator creates campaign
2. Campaign Service validates syntax
3. Safety Engine validates policy
4. Target Resolver queries fleet inventory
5. Immutable target snapshot is written
6. Snapshot hash/device count stored
7. Control version allocated
8. Required approvals recorded
9. Campaign scheduled
10. Outbox emits CAMPAIGN_READY
11. Fanout planner partitions snapshot
12. Work items enter command Kafka
13. Regional dispatchers consume
14. Dispatcher rate-limits delivery
15. Device receives signed command
16. Device performs:
      command-id dedupe
      version fencing
      safety validation
17. Device sends delivery/decision/execution events
18. Gateway authenticates device
19. Events enter telemetry Kafka
20. Stream processor deduplicates
21. Per-device campaign state updated
22. Aggregate delta produced
23. Dashboard updates
24. Late events revise state
25. Reconciliation finds remaining unknowns
26. Final/continuously revised campaign result feeds planner
```

---

# 52. The three interviewer follow-up questions

## 1. How do you distinguish rejection from never receiving the command?

By explicitly separating:

```text
DELIVERY
```

from:

```text
DECISION
```

Rejected:

```text
DELIVERY = DELIVERED
DECISION = REJECTED
reject_reason = ...
```

Never known to have received:

```text
DELIVERY = ATTEMPTED/EXPIRED
DECISION = UNKNOWN
```

Absence of acknowledgment is **unknown**, not rejection.

---

## 2. What if an older command arrives after a newer command?

Every command carries:

```text
command namespace
+
monotonically increasing controlVersion
```

Device durably stores:

```text
latestAppliedVersion[namespace]
```

and requires:

```text
incomingVersion >= latestAllowedVersion
```

A delayed:

```text
v100
```

after:

```text
v101
```

is rejected as:

```text
STALE_VERSION
```

This is **version fencing**.

---

## 3. How do we report progress when many devices are offline?

Never hide unknown devices.

For:

```text
10M targeted
4M measured
3.8M executed
6M unknown
```

report:

```text
Confirmed execution       38% of targeted fleet
Execution among observed  95%
Measurement coverage      40%
Unknown                    60%
```

Any extrapolated result should be explicitly labeled:

```text
estimated
+
confidence interval/confidence level
```

and automatic follow-up commands should be restricted when uncertainty is high.

---

# 53. Interview-level design summary

If I had roughly two minutes to close the interview, I would summarize the architecture like this:

```text
                    ┌────────────────────┐
                    │ Campaign Service   │
                    └─────────┬──────────┘
                              │
                     immutable campaign
                              │
              ┌───────────────▼──────────────┐
              │ Target Resolver / Snapshot   │
              └───────────────┬──────────────┘
                              │
                       partitioned fanout
                              │
                              ▼
                         Command Kafka
                              │
                  ┌───────────┴───────────┐
                  ▼                       ▼
            Dispatcher                Dispatcher
                  │                       │
                  └──────── MQTT ─────────┘
                              │
                           Devices
                              │
             ACK / reject / execute / telemetry
                              │
                              ▼
                        Telemetry Kafka
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
       Device State       Aggregator        Archive
         Store            event-time          S3
             │                │
             │                ▼
             │          Campaign Progress
             │                │
             └───────┐        │
                     ▼        ▼
                    Reconciler
                        │
                        ▼
                  Future Planner
```

The guarantees are:

```text
Immutable campaign + immutable target snapshot
                     +
At-least-once command delivery
                     +
Command-ID idempotency
                     +
Monotonic version fencing
                     +
Authenticated correlated telemetry
                     +
Event-time stream processing
                     +
Revisionable campaign aggregates
                     +
Explicit unknown/offline semantics
                     +
Persistent reconciliation
                     +
Safety limits before automatic follow-up
```

That combination is what makes the system safe even when **networks are unreliable, devices disappear for hours, telemetry arrives late, messages duplicate, and old commands surface after newer ones**.