# Part 3 — Reliable Delivery, Retries, Idempotency, DLQ, and Recovery

> **Context:** This document continues the Trade Event Distribution Service design. It consolidates the concepts covered in Part 3.1–3.4, using the same event, destinations, Kafka partition, and offset throughout.
>
> **Core objective:** No valid event should be silently lost. Delivery failures must be recoverable, duplicate messages must not cause duplicate business effects, and retries must not block the normal low-latency distribution path indefinitely.

---

## Table of contents

1. [System context and running example](#1-system-context-and-running-example)
2. [Delivery Status Store](#2-delivery-status-store)
3. [Retry strategy](#3-retry-strategy)
4. [Idempotency](#4-idempotency)
5. [Database transactions for idempotent processing](#5-database-transactions-for-idempotent-processing)
6. [Dead-letter queues](#6-dead-letter-queues)
7. [Recovery and replay](#7-recovery-and-replay)
8. [Kafka offset commit safety](#8-kafka-offset-commit-safety)
9. [End-to-end failure scenarios](#9-end-to-end-failure-scenarios)
10. [Design trade-offs and interview points](#10-design-trade-offs-and-interview-points)
11. [Quick revision](#11-quick-revision)

---

## 1. System context and running example

We are designing a low-latency Trade Event Distribution Service.

### 1.1 Running event

```json
{
  "eventId": "EVT-1003",
  "rfqId": "RFQ-501",
  "quoteId": "QUOTE-9001",
  "dealerId": "DEALER-25",
  "eventType": "QUOTE_HIT",
  "sequenceNumber": 3,
  "currencyPair": "EUR/USD",
  "quantity": 1500000
}
```

Identifiers have different purposes:

- **EventID** uniquely identifies one business event. It must remain unchanged when that same event is retried or replayed.
- **QuoteID** identifies the quote lifecycle. It is the Kafka message key when we need events for a quote to be ordered in the source topic.
- **RFQID** identifies the request for quote.
- **sequenceNumber** is a business sequence within the quote lifecycle.
- **Kafka offset** identifies a record position within one Kafka partition. It is not the same as the business sequence number.

For our running example:

- Source topic: `trade-events`
- Partition: `P17`
- Offset: `102`
- EventID: `EVT-1003`
- Key: `QUOTE-9001`
- Destinations: Risk, Reporting, Audit

The same event may be delivered to multiple destination topics, for example `risk-events`, `reporting-events`, and `audit-events`.

### 1.2 Delivery status example

Initially, the Distributor attempts all required destinations. The status could become:

| EventID | Destination | Status |
|---|---|---|
| EVT-1003 | Risk | SUCCESS |
| EVT-1003 | Reporting | PENDING |
| EVT-1003 | Audit | PENDING |

This state is useful only if it is stored durably and updated with a crash-safe workflow. An in-memory dictionary is not sufficient.

---

## 2. Delivery Status Store

### 2.1 What problem does it solve?

One event can have multiple destinations, and each destination can succeed or fail independently.

Suppose Risk accepts the event, but Reporting fails. Then the Distributor pod crashes. Kafka may redeliver the source event because its offset was not committed.

Without durable delivery state, the replacement pod may not know that Risk had already succeeded. It may publish the event to Risk again unnecessarily.

A **Delivery Status Store** records the progress of each event-destination pair so a replacement pod or recovery worker can determine what remains to be done.

### 2.2 Example table

A simplified SQL Server table:

```sql
CREATE TABLE EventDeliveryStatus
(
    EventId       VARCHAR(100) NOT NULL,
    Destination   VARCHAR(50)  NOT NULL,
    Status        VARCHAR(20)  NOT NULL,
    RetryCount    INT          NOT NULL DEFAULT 0,
    LastError     VARCHAR(1000) NULL,
    UpdatedAt     DATETIME2    NOT NULL,

    CONSTRAINT PK_EventDeliveryStatus
        PRIMARY KEY (EventId, Destination)
);
```

Example rows:

| EventID | Destination | Status | RetryCount |
|---|---|---|---:|
| EVT-1003 | Risk | SUCCESS | 0 |
| EVT-1003 | Reporting | PENDING | 2 |
| EVT-1003 | Audit | PENDING | 0 |

The primary key is `(EventID, Destination)` because the same event can have a separate delivery outcome for each destination.

Possible states include:

- `PENDING` — delivery is not yet complete.
- `SUCCESS` — destination acknowledged the publish and the success record was persisted.
- `RETRYING` — a retry is scheduled or in progress.
- `DLQ` — normal retry policy was exhausted or intervention is needed.

These states are illustrative; production state transitions should be explicitly defined and concurrency-safe.

### 2.3 What happens after a pod crash?

1. Pod A reads `EVT-1003`.
2. Risk publish succeeds and is recorded as `SUCCESS`.
3. Reporting publish fails and remains `PENDING`.
4. Pod A crashes before committing the source offset.
5. Kafka reassigns the partition to Pod B and redelivers the event.
6. Pod B checks durable delivery state.
7. Pod B skips Risk if its status is reliably `SUCCESS`, retries Reporting, and handles Audit according to its recorded status.

This avoids many unnecessary duplicate publishes.

### 2.4 Important limitation

The status store cannot eliminate every duplicate.

Consider this sequence:

1. Pod A publishes to Risk.
2. Risk's broker acknowledges the publish.
3. Pod A crashes before recording `SUCCESS` in the database.
4. The database still says `PENDING`.
5. Pod B retries Risk.

Risk may receive the same message twice. Therefore:

> The Delivery Status Store tracks delivery progress; destination-side idempotency prevents duplicate business effects.

### 2.5 Performance trade-off

A database read and write for every destination can add latency and load. Our system has a 10 ms distribution target, so we should not automatically add synchronous database round trips to every hot-path operation without measuring their cost.

Possible approaches include:

- Keep the fast path in Kafka and use a durable asynchronous delivery workflow.
- Track only the state required by the recovery guarantees.
- Use a Kafka-based retry/work queue where appropriate.
- Use a database-backed ledger when operational visibility and per-destination state justify the cost.
- Benchmark throughput, p99 latency, database contention, and recovery behavior.

The chosen approach must still ensure that failed work is durably handed off before the source offset is committed.

---

## 3. Retry strategy

### 3.1 Why retry?

A publish failure does not always mean that the event is invalid. It may mean the destination broker is temporarily unavailable, the network timed out, or the service is overloaded.

We should retry transient failures while avoiding endless rapid retries.

| Failure | Typical response |
|---|---|
| Temporary network error or timeout | Retry with backoff |
| Kafka leader election or temporary broker issue | Retry |
| Destination overload / throttling | Back off and retry |
| Invalid event schema | Do not retry unchanged data; send to invalid-event handling |
| Authorization or configuration failure | Alert and investigate; avoid aggressive repeated attempts |
| Persistent failure after retry policy | Persist for DLQ/recovery |

The exact retry policy depends on the failure classification and recovery objectives.

### 3.2 Exponential backoff

Exponential backoff increases the delay after each failed attempt.

A common illustrative formula is:

\[
\text{Delay}_n = \min(\text{BaseDelay}\times 2^{n-1}, \text{MaxDelay})
\]

For example, with a base delay of 100 ms and a maximum delay of 2 seconds:

| Attempt | Illustrative wait before retry |
|---:|---:|
| 1 | 100 ms |
| 2 | 200 ms |
| 3 | 400 ms |
| 4 | 800 ms |
| 5 | 1,600 ms |
| Later attempts | Capped at 2,000 ms |

The exact schedule is policy-specific. The first attempt is usually immediate; the table describes waits after failures.

### 3.3 Add jitter

If thousands of events fail at the same time, they can all retry at the same scheduled moment. This can overload the recovering destination again.

**Jitter** adds a random variation to retry delays, spreading the retry traffic over time.

For example, instead of retrying every event at exactly 400 ms, the system may schedule retries within a range around that delay.

### 3.4 Do not block the Distributor forever

If Reporting is unavailable for two minutes, the main Distributor should not tie up its processing capacity waiting synchronously for Reporting to recover.

Separate:

- **Initial distribution:** make the initial publish attempt promptly.
- **Recovery retry:** persist the unfinished delivery and retry asynchronously.

Example flow:

```text
Distributor
    |
    v
Publish to Reporting
    |
    +---- success ----> Record success
    |
    +---- failure
              |
              v
       Persist retry work
              |
              v
        Retry worker
              |
              v
       Retry with backoff
```

The retry worker may be a separate consumer group or service. A failing destination should not unnecessarily block healthy destinations or all normal event processing.

### 3.5 Durable retry work

A retry record may include:

- `EventID`
- `Destination`
- Original event payload or durable reference
- Retry attempt count
- Next retry time
- Last error
- Creation/update timestamps
- Original source topic, partition, and offset

Do not store the only copy of retry state in process memory. If the pod crashes, the retry must still be discoverable.

**Offset safety rule:** the source offset must not be committed while failed delivery work exists only in memory. Before committing, required work must either be completed or durably handed off to a recovery mechanism.

### 3.6 Maximum retries

Retry indefinitely at full speed is unsafe. Use a bounded retry policy with an explicit recovery path.

Example:

1. Attempt initial delivery.
2. Retry a configured number of times with backoff.
3. If still failing, persist the delivery in a DLQ or failed-delivery state.
4. Alert the appropriate team.
5. Recover or replay once the cause has been resolved.

A retry limit is not permission to discard the event. The failed work must remain durable and visible until it is reconciled.

### 3.7 Kafka retry topics versus a database-backed retry store

**Option A: Kafka retry topics**

Examples: `reporting-retry`, `reporting-dlq`.

Advantages:
- Durable Kafka storage.
- Natural fit for high-throughput asynchronous processing.
- Consumer-based scaling.
- Retry workflow can be isolated from the main Distributor.

Trade-offs:
- Delayed retry requires scheduling logic or tiers of retry topics.
- A retry can change the order in which events are observed.
- Duplicate retry tasks must be handled.
- A destination-specific retry topic may simplify isolation but adds topic and operational complexity.

**Option B: Database-backed retry records**

Advantages:
- Explicit per-destination delivery state.
- Easy to inspect retry count, next retry time, and last error.
- Useful for operations dashboards and manual recovery.

Trade-offs:
- Database reads/writes add load and potentially latency.
- Requires indexing, locking/concurrency control, and worker coordination.
- Must handle duplicate insertion and state transitions safely.

For our high-throughput trading design, Kafka retry topics are a reasonable starting point, with a delivery ledger if the requirements justify it. The choice must be validated against throughput, p99 latency, ordering, and recovery requirements.

### 3.8 Ordering complication during retry

The source topic uses `QuoteID` as its key, which preserves partition order in the source topic. However, asynchronous retries can still cause destination-side reordering.

Example:

1. `QUOTE_UPDATED`, sequence 2, fails to reach Risk.
2. `QUOTE_HIT`, sequence 3, succeeds in reaching Risk.
3. Risk observes sequence 3 before sequence 2.

If strict per-quote ordering is required at Risk, later events for that quote must not overtake an earlier unresolved event. Possible approaches include:

- Pause delivery for the affected quote until the earlier event succeeds.
- Use ordered per-partition processing with retry state that prevents advancing past a failed record.
- Use destination-specific ordered workers and partition by `QuoteID`.
- Validate the business sequence number at the destination and detect gaps.

Partitioning by `QuoteID` alone does not solve ordering if the retry architecture allows a later event to bypass an earlier failed delivery.

---

## 4. Idempotency

### 4.1 The duplicate-message problem

The Distributor publishes `EVT-1003` to the Risk service. Risk updates its database, but the service crashes before the Distributor receives an acknowledgement.

The Distributor cannot know whether the operation happened. It retries the message.

If Risk applies the same business effect twice, the result can be wrong.

For example, if the event increases exposure by ₹10 lakh:
- First processing adds ₹10 lakh.
- Duplicate processing adds another ₹10 lakh.
- The resulting ₹20 lakh increase is incorrect if the event should count once.

### 4.2 Definition

An operation is **idempotent** when repeating it produces the same final business result as executing it once.

In event-driven systems, an idempotent consumer recognizes that it has already applied a particular event's business effect.

Use the stable `EventID` as the deduplication key.

Important distinctions:

- **EventID:** identifies one business event.
- **QuoteID:** identifies the quote lifecycle and can be shared by many events.
- A retry or replay of the same event must preserve the original EventID.
- Generating a new EventID during replay can bypass deduplication.

### 4.3 Processed-events table

```sql
CREATE TABLE ProcessedEvents
(
    EventId     VARCHAR(100) NOT NULL PRIMARY KEY,
    ProcessedAt DATETIME2 NOT NULL
);
```

The unique constraint on `EventId` is essential. A prior read check alone is not sufficient to protect against two workers processing the same event concurrently.

### 4.4 Why this simplified code is unsafe

```csharp
if (await repository.IsProcessedAsync(evt.EventId))
{
    return;
}

await ProcessBusinessLogicAsync(evt);

await repository.MarkProcessedAsync(evt.EventId);
```

Crash window:

1. `IsProcessedAsync` returns false.
2. Business logic succeeds.
3. The service crashes before `MarkProcessedAsync`.
4. The event is retried.
5. The check still returns false.
6. The business logic executes again.

Therefore, the business operation and the processed-event record must be atomic whenever possible.

---

## 5. Database transactions for idempotent processing

### 5.1 Same database transaction

When the processed-event record and the business update are in the same relational database, use one transaction.

Conceptually:

```text
BEGIN TRANSACTION
    |
    v
Insert EventID into ProcessedEvents
    |
    v
Update business state (e.g. risk exposure)
    |
    v
COMMIT
```

If the transaction commits, both changes are durable. If it rolls back, neither change is committed.

The unique constraint on EventID protects against duplicate processing. Production code should recognize a duplicate-key conflict as a duplicate event, not as an arbitrary transient failure.

### 5.2 Illustrative C# with EF Core

```csharp
public async Task ProcessAsync(TradeEvent evt)
{
    await using var tx =
        await db.Database.BeginTransactionAsync();

    try
    {
        // EventId must have a UNIQUE constraint / primary key.
        db.ProcessedEvents.Add(new ProcessedEvent
        {
            EventId = evt.EventId,
            ProcessedAt = DateTime.UtcNow
        });

        // Persist the idempotency record within this transaction.
        await db.SaveChangesAsync();

        // Business effect must be in the same database transaction.
        var exposure = await db.RiskExposures
            .SingleAsync(x => x.QuoteId == evt.QuoteId);

        exposure.Quantity += evt.Quantity;

        await db.SaveChangesAsync();
        await tx.CommitAsync();
    }
    catch
    {
        await tx.RollbackAsync();
        throw;
    }
}
```

This code is illustrative, not complete production code. In particular:

- If inserting EventID fails because it already exists, handle that unique-constraint violation as a duplicate and return successfully.
- Do not swallow other database errors as duplicates.
- The business update and EventID insertion must use the same transaction and database connection/context.
- Protect against concurrent processing using database constraints and appropriate transaction/concurrency controls.
- Validate business rules before applying the update.

### 5.3 What if the business operation and idempotency record are in different systems?

Suppose the Risk service updates SQL Server but stores EventIDs in Redis. These two writes are not atomic under a normal SQL Server transaction.

A crash between the writes can produce inconsistent state:
- Business update succeeds, Redis record fails → duplicate effect on retry.
- Redis record succeeds, business update fails → retry may be incorrectly skipped.

Prefer keeping the processed-event record and business effect in the same database transaction when possible.

If the operation affects an external system, such as submitting a trade through a separate trading platform, consider:
- Transactional outbox.
- Durable workflow/state machine.
- External API's idempotency-key support.
- Reconciliation and compensating actions where appropriate.

A local database transaction cannot make an arbitrary external operation globally atomic.

### 5.4 Kafka transactions and exactly-once processing

Kafka transactions can coordinate supported Kafka input/output and offset operations. They do not automatically make an arbitrary SQL Server update or external business operation atomic with Kafka.

Therefore, do not claim end-to-end exactly-once business effects merely because Kafka transactions are enabled.

A more realistic design combines:
- At-least-once delivery where needed.
- Stable EventIDs.
- Idempotent consumers.
- Atomic business update + processed-event record.
- Durable retry/recovery state.
- Reconciliation for external side effects.

---

## 6. Dead-letter queues

### 6.1 What is a DLQ?

A **dead-letter queue (DLQ)** is a durable place to retain messages or delivery tasks that cannot be completed through the normal retry policy or require manual intervention.

Illustrative Kafka topics:
- `reporting-retry`
- `reporting-dlq`

The names are examples; the design may use destination-specific topics or shared topics with a destination field.

A DLQ is not a place to forget events. It is a controlled holding area for investigation and recovery.

### 6.2 Example failure flow

```text
Distributor
    |
    v
Attempt publish to Reporting
    |
    +---- success ----> Persist SUCCESS
    |
    +---- failure ----> Retry with backoff
                            |
                            v
                      Retry limit reached
                            |
                            v
                 Persist failed delivery in DLQ
                            |
                            v
                  Alert / investigate / recover
```

### 6.3 Useful DLQ metadata

```json
{
  "eventId": "EVT-1003",
  "destination": "Reporting",
  "originalTopic": "trade-events",
  "originalPartition": 17,
  "originalOffset": 102,
  "attemptCount": 5,
  "lastError": "Reporting broker unavailable",
  "firstFailedAt": "2026-10-11T16:20:00Z",
  "lastAttemptAt": "2026-10-11T16:25:00Z",
  "status": "DLQ"
}
```

Also retain the original payload or a durable reference to it. Metadata alone cannot replay the event if the original source record has expired or been deleted.

Useful fields include:
- EventID.
- Destination.
- Original topic, partition, and offset.
- Retry count and timestamps.
- Last error.
- Original payload or durable reference.
- Correlation/trace identifiers, if available.
- Recovery status and audit details.

Avoid logging sensitive trading data unnecessarily; access to DLQ content should be controlled and audited.

### 6.4 Invalid-event topic versus delivery DLQ

These are different paths.

| Situation | Handling |
|---|---|
| Missing required EventID or invalid schema | Invalid-event topic with validation reason |
| Valid event but destination temporarily unavailable | Retry |
| Valid event but destination keeps failing beyond retry policy | Destination DLQ / failed-delivery state |
| Operator replays a previously failed delivery | Audited recovery workflow |

An invalid event may never be eligible for normal routing. A DLQ delivery record represents valid work that could not be completed through the normal delivery process.

---

## 7. Recovery and replay

### 7.1 Destination recovers

Suppose Reporting was down for ten minutes and is now healthy.

A safe recovery workflow is:

1. Operator or recovery worker selects the failed delivery.
2. Confirm the original failure cause is resolved.
3. Replay only the failed destination when possible.
4. Preserve the original EventID.
5. Record the attempt and outcome.
6. Mark delivery `SUCCESS` only after the required acknowledgement and durable status update.
7. Audit who initiated the replay, when, why, and with what outcome.

```text
DLQ / failed-delivery record
           |
           v
      Select event
           |
           v
  Verify destination healthy
           |
           v
 Replay EVT-1003 to Reporting
           |
           v
 Confirm acknowledgement
           |
           v
 Update status + audit recovery
```

Do not blindly resend to every destination if only Reporting failed. Risk may already have received the event.

### 7.2 Why keep the same EventID?

If a replay changes `EVT-1003` to a new ID such as `EVT-2003`, the downstream idempotency check may treat it as a new business event and apply the business effect again.

A replay is a new delivery attempt for the same event, not necessarily a new business event. Preserve EventID and record the replay attempt separately.

### 7.3 Manual replay controls

For a trading system, manual replay should include:
- Authorization and role-based access.
- Audit log of operator, time, reason, event, destination, and result.
- Duplicate-safe execution.
- Rate limits/batches to avoid overwhelming a recovering destination.
- Visibility into pending, successful, and failed replay attempts.
- Controls for ordering and business-sequence gaps.

Replay should be a controlled operational workflow, not an ad hoc script that blindly republishes records.

---

## 8. Kafka offset commit safety

### 8.1 What does the committed offset mean?

Kafka stores the consumer group's committed offset. The consumer application decides when to commit it, and the Kafka client sends that commit.

Under the common Kafka consumer API convention, the committed offset is the **next offset to read**.

For our example:
- Current record: partition `P17`, offset `102`.
- Once offset 102 is safely handled, commit offset `103`.
- A restarted consumer resumes at 103.

### 8.2 Crash cases

**Case A — crash before completion and before committing 103**

1. Pod A consumes offset 102.
2. It crashes before the required work is safely completed and before committing 103.
3. Kafka reassigns the partition to Pod B.
4. Pod B resumes from the last committed offset, potentially 102.
5. The event is processed again.

This is expected at-least-once behavior.

**Case B — work completes and commit 103 succeeds**

1. Pod A completes the required work.
2. It commits offset 103.
3. Pod A crashes.
4. Pod B resumes at offset 103.

Offset 102 is not automatically redelivered to that consumer group.

### 8.3 Dangerous case: commit before durable handoff

Unsafe sequence:

1. Reporting delivery fails.
2. Failure is held only in pod memory.
3. The consumer commits offset 103.
4. Pod crashes.
5. Kafka resumes at 103, and the in-memory failure state is gone.

The failed delivery may be lost.

Safe principle:

> Do not commit the source offset until required delivery work has completed or has been durably handed off to a recoverable workflow.

### 8.4 Database and Kafka offset are separate systems

Suppose the Distributor inserts a retry record into SQL Server and then commits the Kafka offset.

A crash after the database insert but before the offset commit can cause the source event to be consumed again and the retry record to be inserted again.

The reverse ordering is more dangerous: if the offset commits before the retry record is durable, the failed delivery can be lost.

Design implications:
- Make retry-task insertion idempotent, e.g. unique key `(EventID, Destination)`.
- Make state transitions concurrency-safe.
- Ensure the durable handoff is complete before source offset commit.
- Handle reprocessing after a failed offset commit.
- Consider an outbox or another durable workflow if the coordination requirements demand it.

A database transaction and a Kafka offset commit are not automatically atomic together.

---

## 9. End-to-end failure scenarios

### 9.1 Scenario A — Risk succeeds, Reporting fails, Pod A crashes

Starting state:
- Event: `EVT-1003`
- Source: `trade-events`, partition `P17`, offset `102`
- Required destinations: Risk, Reporting, Audit

Sequence:
1. Pod A consumes offset 102.
2. Risk publish succeeds.
3. Delivery status for Risk is durably set to `SUCCESS`.
4. Reporting publish fails.
5. Failed Reporting delivery is durably recorded as retry work.
6. Audit is handled according to the chosen delivery policy.
7. Only after all required deliveries are completed or durably handed off, the consumer commits offset 103.
8. Pod A crashes.
9. The retry worker continues processing Reporting independently.
10. If Reporting recovers, the worker retries the delivery with the original EventID.
11. Reporting success is durably recorded.

If Pod A crashes before step 7, Kafka can redeliver offset 102. The status store and idempotent retry-task insertion help determine what work remains. Destination idempotency remains necessary because a publish can succeed before its success record is written.

### 9.2 Scenario B — destination ACK succeeds, but status update fails

1. Pod A publishes to Risk.
2. Kafka acknowledges the publish.
3. Pod A crashes before the delivery status update is persisted.
4. The status store still says `PENDING`.
5. A replacement worker may publish to Risk again.

This is why the status store alone cannot guarantee no duplicate messages. Risk must deduplicate using EventID, ideally atomically with the business update.

### 9.3 Scenario C — destination is unavailable for a long time

1. Initial publish to Reporting fails.
2. Retry work is persisted.
3. Retry worker uses exponential backoff and jitter.
4. The main Distributor continues handling unrelated events where the ordering and capacity design permits.
5. Retry policy reaches its configured limit.
6. Delivery moves to a durable failed-delivery state or DLQ.
7. An alert is raised.
8. After the cause is fixed, an operator or recovery worker replays the failed delivery.
9. The original EventID is retained, and the outcome is audited.

### 9.4 Scenario D — duplicate delivery to Risk

1. Risk receives `EVT-1003`.
2. Risk inserts the EventID and updates exposure in the same transaction.
3. The event is delivered again.
4. The unique constraint detects the already processed EventID.
5. The duplicate is acknowledged/handled without applying the exposure update again.

The exact duplicate-acknowledgement behavior should be implemented carefully: a known duplicate is normally treated as successfully handled, while unrelated database errors must still surface as failures.

---

## 10. Design trade-offs and interview points

### 10.1 What does “exactly once” mean?

Be precise in interviews:
- A message may be delivered more than once.
- A consumer can make the business effect effectively once by using a stable EventID and atomic idempotency handling.
- Kafka transactions can provide exactly-once semantics for supported Kafka workflows.
- Arbitrary external systems require their own idempotency or reconciliation strategy.

Avoid promising global exactly-once processing across Kafka, SQL Server, and external trading systems without a concrete atomicity mechanism.

### 10.2 What should the 10 ms target mean?

Clarify whether 10 ms means:
- End-to-end latency from producer to final business processing.
- Ingestion acknowledgement latency.
- Distributor processing latency.
- p99 latency or a hard maximum.

Durable database writes, multiple destination publishes, retries, and synchronous downstream dependencies can all affect latency. A resilient design should keep slow recovery work off the main path when requirements permit, while preserving durable handoff guarantees.

### 10.3 Delivery ledger versus Kafka retry topics

A ledger provides operational visibility and explicit per-destination state. Retry topics can scale naturally with Kafka and avoid a database query for every retry. Some designs use both:
- Kafka for event transport and retry scheduling.
- A compact durable ledger for delivery status, operations, and replay audit.

Do not add a ledger just because it sounds safer. Explain which recovery requirement it solves and measure its impact.

### 10.4 Ordering versus availability

Strict ordering can force later events to wait behind an earlier failed event for the same quote. That reduces availability or throughput for that quote during an outage.

The system must decide whether the business requires:
- Strict per-quote ordering at every destination.
- Ordering only in the source stream.
- Business sequence validation with gap detection.
- Relaxed ordering for specific destinations.

State the chosen requirement and enforce it explicitly.

### 10.5 Useful monitoring and alerts

Monitor at least:
- Publish failures by destination.
- Retry count and retry age.
- Number of pending deliveries.
- DLQ size and oldest-message age.
- Consumer lag by partition and group.
- Time from source ingestion to destination acknowledgement.
- Duplicate EventID detections.
- Recovery/replay success and failure.
- p50, p95, and p99 latency.
- Per-destination throughput and error rate.

Alert on persistent delivery failures, growing retry backlog, increasing DLQ size, and latency/lag breaches.

---

## 11. Quick revision

### 11.1 One-line definitions

| Concept | Short definition |
|---|---|
| Delivery Status Store | Durable per-event, per-destination delivery progress |
| Retry | Reattempt a transient failure |
| Exponential backoff | Increase the wait between failed attempts |
| Jitter | Randomize retry delay to spread load |
| Idempotency | Repeated processing does not repeat the business effect |
| ProcessedEvents table | Durable record of EventIDs already applied |
| Database transaction | Atomically commits the EventID record and business update |
| DLQ | Durable holding area for deliveries that need intervention/recovery |
| Replay | Controlled reattempt of a previously failed delivery |
| Kafka committed offset | Position from which the consumer group resumes; commonly the next offset to read |
| At-least-once delivery | Redelivery may happen, so consumers must tolerate duplicates |
| Exactly-once business effect | Achieved within a defined boundary using deduplication and atomic business updates; not automatically global |

### 11.2 Five rules to remember

1. **Never rely on in-memory retry state.** Persist unfinished work.
2. **Never commit the source offset before work is complete or durably handed off.**
3. **Use a stable EventID.** Keep it unchanged across retries and replay.
4. **Make the idempotency record and business update atomic** when they share a database.
5. **A DLQ is for recovery, not deletion.** Keep failed work visible, auditable, and replayable.

### 11.3 Interview answer in one paragraph

“I would use a durable per-destination delivery workflow so partial failures can be recovered without blindly resending to every destination. Transient failures would be retried asynchronously using exponential backoff and jitter, with bounded attempts and a DLQ for persistent failures. I would not commit a Kafka source offset until required delivery work had completed or had been durably handed off. Since redelivery is possible, every event would have a stable EventID, and downstream consumers would deduplicate it using a unique constraint and an atomic transaction that combines the processed-event record with the business update. For external side effects, I would use the external system's idempotency support, an outbox/workflow, or reconciliation rather than claiming global exactly-once semantics. I would also preserve per-quote ordering where the business requires it, audit manual replay, and monitor retries, DLQ age, consumer lag, and p99 latency.”

---

## 12. Part 3.5 — End-to-end failure and recovery flow

This section connects the Delivery Status Store, retries, idempotency, database transactions, DLQ, and Kafka offset management into one complete flow.

### 12.1 Starting state

Our event is:

- EventID: `EVT-1003`
- QuoteID: `QUOTE-9001`
- Source topic: `trade-events`
- Source partition: `P17`
- Source offset: `102`
- Required destinations: Risk, Reporting, Audit

The source event is keyed by `QuoteID` so events for that quote are assigned to the same source partition. This supports source-topic ordering, but destination retries must still be designed so a later event cannot overtake an earlier one when strict destination ordering is required.

### 12.2 Normal processing flow

```text
Producer
   |
   v
trade-events (P17, offset 102)
   |
   v
Distributor Pod A
   |
   +----> Risk      ---- ACK ----> persist Risk = SUCCESS
   |
   +----> Reporting ---- ACK ----> persist Reporting = SUCCESS
   |
   +----> Audit     ---- ACK ----> persist Audit = SUCCESS
   |
   v
All required deliveries complete
   |
   v
Commit source offset 103
```

The source offset is committed only after the required work has completed and the delivery state needed for recovery is durable. The exact implementation must account for the fact that destination publishes, database updates, and Kafka offset commits are not one global atomic transaction.

### 12.3 Failure flow: Risk succeeds, Reporting fails

Assume Risk accepts the event, but Reporting is unavailable.

```text
trade-events P17 / offset 102
             |
             v
       Distributor A
             |
      +------+------+
      |             |
      v             v
   Risk          Reporting
      |             |
    ACK          publish fails
      |             |
      v             v
Persist         Persist retry work
SUCCESS         durably
                    |
                    v
              Retry worker
```

The delivery state may look like this:

| EventID | Destination | State |
|---|---|---|
| EVT-1003 | Risk | SUCCESS |
| EVT-1003 | Reporting | RETRYING / PENDING |
| EVT-1003 | Audit | Depends on whether it has been delivered or durably handed off |

The source offset must not be committed merely because the Distributor has tried the publishes. It may commit once every required delivery has either succeeded or has been safely handed off to a durable recovery workflow, according to the system's explicit delivery contract.

### 12.4 Pod crashes before source offset commit

Suppose Pod A crashes before it commits offset 103.

1. Kafka detects the consumer failure and reassigns partition P17 to another consumer in the same consumer group.
2. Pod B resumes from the last committed offset. If offset 103 was not committed, offset 102 may be delivered again.
3. Pod B reads `EVT-1003` and checks durable delivery state and retry-work records.
4. It does not blindly assume every destination failed or every destination succeeded.
5. It skips destinations whose successful delivery is durably recorded, retries or resumes pending work, and uses idempotent insertion for retry tasks.
6. It safely completes or durably hands off the required work before committing the source offset.

A redelivery is expected in an at-least-once design. Correctness comes from durable state, idempotent retry-task creation, and destination-side idempotency—not from assuming Kafka will never redeliver.

### 12.5 Crash window: publish succeeded but status update did not

This is a critical failure window:

1. Pod A publishes `EVT-1003` to Risk.
2. Risk's broker acknowledges the message.
3. Pod A crashes before writing `Risk = SUCCESS` to the delivery ledger.
4. The ledger still says `PENDING`.
5. Pod B may publish the same event to Risk again.

The status ledger cannot prove that Risk did not receive the message. Therefore, Risk must deduplicate by EventID and atomically commit its processed-event record with the business update.

```text
Risk receives EVT-1003
        |
        v
BEGIN DB TRANSACTION
        |
        +--> Insert EventID (unique constraint)
        |
        +--> Apply risk exposure update
        |
        v
COMMIT
```

If the same EventID arrives again, the unique constraint identifies it as a duplicate and the business effect is not applied twice. If the transaction fails, both the EventID record and business update roll back together.

### 12.6 Retry worker succeeds later

If Reporting is unavailable for two minutes, its retry work remains durable while the normal Distributor continues processing unrelated events where ordering and capacity constraints allow.

When Reporting recovers:

1. The retry worker claims an eligible retry task safely so multiple workers do not race without coordination.
2. It checks that the destination is healthy enough to retry.
3. It republishes the original event to Reporting using the original EventID and appropriate destination key.
4. If the publish succeeds and the acknowledgement is received, it records the delivery success durably.
5. It records the attempt result for monitoring and audit.
6. The retry task is marked complete only after the success state is durable.

If the worker crashes after the destination accepted the message but before the success record is saved, it may retry again. Destination-side idempotency remains required.

### 12.7 Retry limit reached: DLQ and manual recovery

If Reporting keeps failing after the configured retry policy:

1. The retry workflow durably records the failed delivery in a DLQ or equivalent failed-delivery state.
2. The record retains EventID, destination, source topic/partition/offset, retry count, last error, timestamps, and the original payload or a durable reference to it.
3. An alert is raised and an operator investigates the cause.
4. Once the cause is resolved, an authorized operator or recovery worker replays only the failed Reporting delivery when appropriate.
5. The replay keeps EventID `EVT-1003`, records a separate replay attempt/audit entry, and applies rate limits to avoid flooding a recovering service.
6. After a confirmed publish acknowledgement, the delivery state is updated to `SUCCESS`.

A DLQ is not a discard bin. The event remains unresolved until the delivery is recovered or handled under an explicitly approved business policy.

### 12.8 The two important commit orderings

**Safer ordering:**

```text
Persist retry task / delivery state durably
                  |
                  v
Commit Kafka source offset
```

If the pod crashes after the durable write but before the Kafka offset commit, Kafka may redeliver the source event. The system must make the retry-task write idempotent, for example with a unique key `(EventID, Destination)`.

**Unsafe ordering:**

```text
Commit Kafka source offset
                  |
                  v
Attempt to persist retry task
```

If the offset commit succeeds and the pod crashes before the retry task is stored, Kafka will resume after the source event and the failed delivery may be lost.

A SQL transaction and a Kafka offset commit are separate operations unless a specifically designed coordination pattern is used. Never imply they are automatically atomic.

### 12.9 End-to-end state transition summary

| Situation | Durable state/action | Source offset action |
|---|---|---|
| All required destinations succeeded | Persist each required success | Commit next offset after required state is durable |
| One destination temporarily fails | Persist retry task and next retry time | Commit only after durable handoff is confirmed and contract permits it |
| Pod crashes before safe handoff | No safe completion can be assumed | Do not assume the offset is committed; expect redelivery if uncommitted |
| Publish succeeds but success update is lost | Ledger may remain pending | Duplicate publish is possible; destination idempotency protects business effect |
| Retry policy exhausted | Persist DLQ/failed-delivery record and alert | Commit only after failed work is durably recoverable under the design |
| Manual replay succeeds | Persist success and audit record | Source offset is not normally rewound just to replay one destination |

### 12.10 Interview explanation

“I would treat delivery as durable per-event, per-destination work. The Distributor publishes to the required destinations and records progress. Transient failures are persisted for asynchronous retry with exponential backoff and jitter. The source offset is committed only after required work has completed or has been durably handed off to a recoverable workflow. If a pod crashes before the offset commit, Kafka may redeliver the event, so retry-task creation must be idempotent. A publish may have succeeded even if its status update was lost, so downstream services deduplicate using the stable EventID and atomically combine the processed-event record with the business update. After bounded retries, failed deliveries enter a durable DLQ for audited recovery. I would also preserve required per-quote ordering and monitor consumer lag, pending delivery age, retry backlog, DLQ growth, and p99 latency.”

---

**End of Part 3.**

The design is only complete once the retry store, delivery-status transitions, offset commits, and replay workflow have been made crash-safe together. The examples in this document illustrate the core mechanisms; production implementation needs explicit state transitions, concurrency control, schema evolution, retention policies, security, and failure testing.
