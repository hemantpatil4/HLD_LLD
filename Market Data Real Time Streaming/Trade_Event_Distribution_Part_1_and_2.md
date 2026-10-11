# Real-Time Trade Event Distribution — Parts 1 and 2
## Plain-language interview notes with diagrams and questions covered

## 1. Goal and requirements

We are designing a service that reads trading events, decides where each event should go, and publishes it to the required downstream systems.

Requirements discussed:
- Target of 10 ms for the Distributor path. We still need to clarify whether this is a hard maximum or a percentile such as p99, and exactly where timing starts/stops.
- No silent loss of valid events.
- Preserve ordering for events belonging to the same quote lifecycle.
- Keep valid events available for 60 days.
- Support controlled manual replay for recovery.
- Record invalid events and the reason they were rejected.
- Handle retries and duplicates safely.
- Account for uneven load and hot quote keys.

## 2. Architecture

```text
Trading Event Producer
        |
        | HTTPS POST /events (working assumption)
        v
+--------------------------+
| Event Ingestion Service  |
| validate schema/fields   |
+--------------------------+
       | valid       | invalid
       v             v
  trade-events   invalid-trade-events
       |
       v
+-----------------------------------+
| Trade Distribution Service        |
| multiple pods, one consumer group |
| read -> filter -> route -> commit |
+-----------------------------------+
   |             |              |
   v             v              v
risk-events  reporting-events  settlement-events

A separate History Writer consumes trade-events and stores
valid events in a 60-day archive.
```

### What each block does
- **Trading Event Producer:** existing system that creates the event.
- **Event Ingestion Service:** independently deployed application. Validates payload and publishes valid events to Kafka; invalid ones go to a durable invalid-event topic with a reason.
- **Kafka:** durable buffer and event log. It provides partitions, independent consumer groups, and replay while data remains retained. It is not automatically the 60-day archive.
- **Trade Distribution Service:** independently deployed application with multiple pods. Reads events, evaluates routing rules, publishes to destination topics, and commits the source offset only when required work is safely handled.
- **Downstream services:** Risk, Reporting, Settlement, and Audit perform their own business processing.
- **History Writer:** separate consumer group that stores all valid source events, whether or not the Distributor's filters match them.

## 3. The same event used throughout

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

| Field | Meaning |
|---|---|
| `eventId` | Unique identifier for this specific event; used for deduplication. |
| `rfqId` | Identifies the request for quote. |
| `quoteId` | Identifies the quote lifecycle; useful as the Kafka key for quote ordering. |
| `dealerId` | Identifies the dealer. |
| `eventType` | What happened, e.g. `QUOTE_HIT`. |
| `sequenceNumber` | Business sequence within this quote. |
| Kafka offset | Position of a record inside one Kafka partition. Not the same as `sequenceNumber`. |

Example lifecycle:
```text
QUOTE-9001
  sequence 1: QUOTE_SUBMITTED
  sequence 2: QUOTE_UPDATED
  sequence 3: QUOTE_HIT
  sequence 4: TRADE_EXECUTED
```
Each event gets a different `EventID`, but all events in this lifecycle share the same `QuoteID`.

## 4. Topics and filter logic

| Topic | Purpose |
|---|---|
| `trade-events` | Original valid trading events. |
| `risk-events` | Events routed to Risk. |
| `reporting-events` | Events routed to Reporting. |
| `settlement-events` | Events routed to Settlement. |
| `audit-events` | Events routed to Audit. |
| `invalid-trade-events` | Invalid event payloads plus rejection reason. |

Illustrative routing:
| Event type | Destinations |
|---|---|
| `QUOTE_SUBMITTED` | Reporting |
| `QUOTE_UPDATED` | Reporting, Audit |
| `QUOTE_HIT` | Risk, Reporting, Audit |
| `TRADE_EXECUTED` | Risk, Reporting, Settlement, Audit |

The routing/filter logic lives in the Trade Distribution Service. It can evaluate event type and other attributes such as product, dealer, currency pair, or status. The Distributor decides **where the event goes**; downstream services decide **what business action to perform**.

```text
trade-events -> Distributor -> evaluate rules
                              |-> risk-events
                              |-> reporting-events
                              |-> audit-events
```

## 5. Partitions, keys, and ordering

Working example:
```text
Topic: trade-events
Key: QUOTE-9001
Partition: P17
Offset: 102
EventID: EVT-1003
```

We use `QuoteID` as the Kafka key when events for the same quote must stay ordered. Kafka maps a key to a partition. Events with the same key normally go to the same partition under a stable partitioning setup.

```text
trade-events (64 illustrative partitions)
 P0: events for many quote keys
 P1: events for many quote keys
 ...
 P17: EVT-1003 at offset 102
 ...
 P63: events for many quote keys
```

64 partitions and replication factor 3 are example assumptions, not final capacity numbers. Measure event rate, payload size, peak bursts, broker throughput, consumer throughput, retention, and latency before choosing production settings.

A hot quote is an important limit: adding partitions spreads different quote IDs, but cannot spread one quote's ordered event stream across multiple partitions without changing the ordering design.

## 6. Consumer groups and pods

All Distributor pods use the same group ID, for example `trade-distributor-group`.

```text
trade-events: 64 partitions
  Pod A / consumer A -> a subset of partitions
  Pod B / consumer B -> a subset of partitions
  Pod C / consumer C -> a subset of partitions
```

- Kafka assigns partitions among consumers in the same group.
- One partition is assigned to at most one active consumer in that group at a time.
- One consumer can own multiple partitions.
- If a pod crashes, Kafka detects membership change and reassigns its partitions after a rebalance.
- If consumers outnumber partitions, some consumers remain idle for that topic.

A consumer group is not a service by itself. It is Kafka's coordination mechanism; the service is the application running in pods.

Can the Distributor handle other streams? Technically, an application can consume multiple topics or groups. For this design, start with one Trade Distribution Service consuming `trade-events`. Separate services/groups are often cleaner if market data or audit has different latency, ordering, throughput, or failure requirements.

## 7. Offsets: the basic rule

A partition is an ordered log. For `P17`:

```text
offset 100 -> event A
offset 101 -> event B
offset 102 -> EVT-1003
offset 103 -> event D
```

Kafka stores the consumer group's committed position. The Distributor application chooses when to commit; its Kafka client sends the commit request.

In the usual consumer API convention, the committed offset is the **next offset to read**:
- Commit `102` -> resume at 102.
- Commit `103` -> offset 102 is considered handled; resume at 103.

Reading an event is not the same as completing it.

### Question: “If 102 succeeds, the consumer in the pod commits, right?”
Yes. If event 102 is fully handled and the required destination outcomes are safely recorded, the consumer commits **103**, not 102.

```text
Read offset 102
     -> evaluate filters
     -> publish to required destinations
     -> safely handle required outcomes
     -> commit offset 103
```

Do not commit 103 merely because the event was fetched.

## 8. Crash case 1: offset 102 was not completed

Assume the last committed offset is 102. Pod A reads offset 102, then crashes before finishing and before committing 103.

```text
Pod A reads offset 102 -> crashes
             |
Kafka rebalances P17
             |
Pod B takes P17 -> resumes at committed offset 102
             |
Pod B processes EVT-1003 again
```

This is expected. Kafka does not treat fetching a record as proof it was processed. Retrying can create duplicates, so the application must tolerate them.

## 9. Crash case 2: offset 102 completed and committed

Pod A finishes required work, commits 103, then crashes.

```text
Pod A completes offset 102 -> commits 103 -> crashes
                                         |
Kafka rebalances P17                     |
                                         v
Pod B takes P17 -> resumes at offset 103
```

| Situation | Committed offset | Replacement resumes at |
|---|---:|---:|
| 102 not completed | 102 | 102 |
| 102 completed and committed | 103 | 103 |

The commit is a position marker, not proof that every downstream business operation succeeded unless the application has correctly tied committing to its durable completion rules.

## 10. Part 2: one destination succeeds and another fails

The Distributor reads `EVT-1003` from `trade-events`, P17, offset 102. It must publish to Risk and Reporting.

```text
                   EVT-1003
                       |
                 Distributor A
                   /       \
                  v         v
            risk-events  reporting-events
              SUCCESS       FAILED
```

If Pod A crashes before committing 103, Pod B resumes from offset 102 and processes the event again. The source offset and each destination's publish acknowledgement are separate operations.

### Option A: retry the entire event
- Risk receives a second publish attempt.
- Reporting is retried.
- Risk may see a duplicate.

### Option B: retry only the failed destination
- If durable state says Risk succeeded, skip it.
- Retry Reporting.
- Requires durable delivery tracking.

## 11. EventID and duplicate handling

Yes: treat `EventID` as the unique identifier for one business event.

| EventID | QuoteID | Event |
|---|---|---|
| `EVT-1001` | `QUOTE-9001` | `QUOTE_SUBMITTED` |
| `EVT-1002` | `QUOTE-9001` | `QUOTE_UPDATED` |
| `EVT-1003` | `QUOTE-9001` | `QUOTE_HIT` |

The producer must create the ID once and preserve it across retries. If every retry creates a new ID, deduplication fails.

Illustrative downstream logic:
```csharp
if (await repository.IsProcessedAsync(event.EventId))
{
    return; // Already applied; ignore duplicate
}

await ProcessBusinessLogicAsync(event);
await repository.MarkProcessedAsync(event.EventId);
```

This is simplified pseudocode. In production, the business update and the processed-ID record should be atomic, commonly in the same database transaction. Otherwise, the service could crash between them and apply the effect twice.

`EventID` does not stop duplicate messages from being sent. It lets the receiver recognize them and avoid applying the business effect twice.

## 12. Durable delivery status

To retry only failed destinations, the Distributor can keep durable per-destination state:

| EventID | Destination | Status |
|---|---|---|
| `EVT-1003` | Risk | `SUCCESS` |
| `EVT-1003` | Reporting | `PENDING` |
| `EVT-1003` | Audit | `PENDING` |

```text
Distributor -> Delivery Status Store
                EVT-1003 / Risk      / SUCCESS
                EVT-1003 / Reporting / PENDING
                EVT-1003 / Audit     / PENDING
```

After a crash, Pod B reads the status:
- Risk `SUCCESS` -> skip redundant send if the record is trustworthy.
- Reporting `PENDING` -> retry.
- Audit `PENDING` -> retry.

Important failure window: Risk may accept the message and Pod A may crash before recording `SUCCESS`. Pod B may then retry Risk. Therefore, durable delivery status helps but does not remove every duplicate. Downstream idempotency is still required.

Kafka transactions can atomically coordinate certain Kafka input/output/offset workflows, but they do not automatically make arbitrary database writes and external business effects globally atomic. We have not selected a final production recovery mechanism yet.

## 13. 60-day history, invalid events, and replay

```text
                 trade-events
                  /        \
                 v          v
           Distributor   History Writer
                |             |
                v             v
         destination topics  60-day archive
                                  |
                                  v
                         controlled manual replay
```

The History Writer uses its own consumer group and stores every valid event, independently of routing filters. Kafka retention must cover the expected outage/recovery window; the separate archive is our 60-day source of long-term history.

Invalid events go to `invalid-trade-events` with reason and metadata. The invalid topic needs its own retention, monitoring, access control, and replay/correction process.

Replay is an operational workflow, not normal live processing. Preserve the original `EventID` when replaying the same business event and make downstream processing idempotent.

## 14. Interview summary

“I would validate incoming events and publish valid ones to `trade-events`. A horizontally scaled Distributor consumer group evaluates routing rules and publishes to destination topics. I would key by `QuoteID` for quote-level ordering and use `EventID` for deduplication. The source offset is committed only after the required processing outcomes are safely handled. Because a crash can happen after one destination succeeds but before another succeeds, I would use durable delivery state where appropriate and require idempotent downstream processing. A separate History Writer stores events for 60 days, and replay is a controlled operational workflow.”

## 15. Questions covered and answers, in plain language

1. **Does the Distributor only process `trade-events`?** For our initial design, yes. It can technically consume more topics/groups, but streams with different operational needs are often split into separate services/groups.
2. **Does event-type filtering live in the Distributor?** Yes. It evaluates event type and possibly other fields to choose destination topics.
3. **Who manages offsets when a pod crashes?** Kafka stores committed offsets; the Distributor application chooses when to commit through its Kafka client.
4. **If 102 was not completed, where does the replacement resume?** At 102, because that is the last committed position.
5. **If 102 was completed and offset 103 committed, where does it resume?** At 103.
6. **If Risk succeeds but Reporting fails?** A crash can cause the event to be retried. Risk may see a duplicate unless it deduplicates by `EventID`.
7. **Can `EventID` be unique per event?** Yes. Keep it unchanged across retries of the same event. `QuoteID` stays the same across one quote lifecycle; `EventID` changes per event.
8. **Can we retry only the failed destination?** Yes, if durable delivery state records which destinations succeeded. Still use idempotency because a crash can happen between destination success and recording that success.

## 16. Open questions for later parts

- Does 10 ms mean hard max or p99, and does it include destination broker acknowledgement?
- What throughput and event size determine partition count?
- What retry/backoff/DLQ policy should be used?
- What durable delivery pattern is best for our exact reliability requirements?
- How should hot quote keys be handled?
- How do we audit and safely execute replay?
