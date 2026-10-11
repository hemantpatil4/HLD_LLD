# Trade Event Distribution Service — Problem Statement and Requirements

**Document purpose:** A single source of truth for the problem statement, functional requirements, non-functional requirements, and decisions still to be clarified for the system design.

**Status:** Working design baseline. Requirements marked as open must be confirmed before treating the design as final.

---

## 1. Problem statement

Design a reliable, low-latency Trade Event Distribution Service for a trading platform.

The system receives trade and quote lifecycle events from an existing producer, validates them, and distributes each valid event to the appropriate downstream systems based on event type and routing rules. Events may need to reach one or multiple destinations, such as Risk, Reporting, Settlement, and Audit.

The system must preserve required ordering, avoid silent event loss, tolerate pod, broker, and downstream failures, and support controlled manual replay for disaster recovery, downstream outages, and rollback scenarios. Every valid event must remain recoverable for 60 days. Invalid events must be durably recorded with the reason they were rejected.

The initial latency target is 10 ms for the distribution path, but the exact measurement boundary and percentile/hard-limit interpretation must be clarified. The architecture must balance low latency with durable acknowledgement, retry safety, historical recovery, idempotency, and observability.

### 1.1 High-level flow

```text
Existing Event Producer
         |
         v
Event Ingestion Service
         |
         +---- invalid ----> Invalid Event Topic / Durable Rejection Record
         |
         v
Kafka: trade-events
         |
         +------------------------+
         |                        |
         v                        v
Distributor Consumer Group     History Writer Consumer Group
         |                        |
         v                        v
Routing rules                 Durable 60-day archive
         |
         +------> Risk
         +------> Reporting
         +------> Settlement
         +------> Audit
```

The Distributor and History Writer use separate Kafka consumer groups. The Distributor routes live events; the History Writer independently archives valid events. Their offsets and recovery progress are independent.

---

## 2. Functional requirements (FR)

Functional requirements describe **what the system must do**.

| ID | Requirement | Details / acceptance intent |
|---|---|---|
| FR-01 | Ingest trade events | Accept events from the existing producer through the agreed ingestion interface. |
| FR-02 | Validate events | Validate schema and required fields. Invalid events must be durably recorded with a rejection reason. |
| FR-03 | Route valid events | Use event type and applicable attributes/rules to route events to destinations such as Risk, Reporting, Settlement, and Audit. |
| FR-04 | Preserve required ordering | Preserve ordering for the agreed scope, currently modeled using `QuoteID` as the Kafka key. The exact business scope must be confirmed. |
| FR-05 | Retain valid events for 60 days | Maintain a durable archive that supports historical recovery for the full retention period, independently of live distribution. |
| FR-06 | Support manual replay | Allow authorized operators to replay events for disaster recovery, downstream outages, and rollback recovery. |
| FR-07 | Retry failed deliveries | Retry transient failures, track progress, and preserve persistently failed deliveries for recovery. |
| FR-08 | Prevent duplicate business effects | Use stable EventIDs and idempotent destination processing so retries/replays do not apply a business effect twice. |
| FR-09 | Prevent silent loss | Track acknowledgements, durable handoffs, failed deliveries, archive progress, and recovery outcomes. |
| FR-10 | Provide traceability | Record validation failures, delivery status, retry history, and replay activity. |

### 2.1 Event identity and ordering fields

Example event:

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

- **EventID** uniquely identifies one business event and remains unchanged across retries and replay.
- **QuoteID** identifies the quote lifecycle and is the current candidate Kafka key for ordering.
- **RFQID** identifies the request for quote.
- **DealerID** identifies the dealer associated with the event.
- **sequenceNumber** represents the business sequence within the relevant lifecycle.
- **Kafka offset** identifies a record position within a particular partition; it is not the same as `sequenceNumber`.

---

## 3. Non-functional requirements (NFR)

Non-functional requirements describe **how well the system must operate**.

| ID | Requirement | Details / acceptance intent |
|---|---|---|
| NFR-01 | Low latency | Target 10 ms for the distribution path. Confirm exact start/end boundary and whether this is p99 or a hard maximum. |
| NFR-02 | Reliability and durability | No silent loss of valid events. Acknowledge ingestion only after the agreed durable Kafka write succeeds. |
| NFR-03 | 60-day retention | Every valid event must remain recoverable for 60 days, even if Kafka's own retention is shorter than the recovery requirement. |
| NFR-04 | Scalability | Support increasing throughput using Kafka partitions and horizontally scaled consumer pods. |
| NFR-05 | Fault tolerance | Recover from pod crashes, broker failures, and downstream outages without losing durable work. |
| NFR-06 | Ordering | Preserve the confirmed ordering scope, especially for events belonging to the same quote or dealer. |
| NFR-07 | Idempotency | Redelivery and manual replay must not duplicate business effects. |
| NFR-08 | Recoverability | Support controlled replay, checkpoints, and visible recovery progress. |
| NFR-09 | Observability | Monitor latency, consumer lag, archive lag, retry backlog, DLQ size/age, failures, duplicate detections, and replay outcomes. |
| NFR-10 | Security and auditability | Restrict access to event data and replay operations; audit manual recovery actions. |

---

## 4. Key design constraints and assumptions

These are the current working assumptions used to develop the architecture:

1. **Ingestion:** The existing producer sends HTTPS requests to an Event Ingestion Service. This is a working assumption; direct producer-to-Kafka publishing could be considered if the existing environment supports it.
2. **Kafka source topic:** Valid events are published to `trade-events`.
3. **Durable ingestion acknowledgement:** Ingestion responds successfully only after Kafka acknowledges the record using the selected durability configuration. Example settings such as replication factor 3, `acks=all`, and an appropriate minimum in-sync replica count must be capacity-tested and finalized.
4. **Ordering key:** `QuoteID` is the current candidate Kafka key. This preserves order for the same key in a partition, but does not by itself guarantee ordering across different topics or through asynchronous retries.
5. **Consumer groups:** Distributor and History Writer are independent consumer groups with separate offsets.
6. **Distributor:** Runs as multiple pods, evaluates routing rules, and publishes to destination topics.
7. **History Writer:** Runs as separately scalable pods and archives valid events to durable storage for 60 days.
8. **Retry/recovery:** Failed delivery work must be durably completed or handed off before the source offset is committed.
9. **Idempotency:** EventID remains stable across retries/replay. Destinations should atomically record processed EventIDs with the corresponding business update when both are in the same database.
10. **Replay:** Replay is a controlled, auditable operational workflow, not an untracked manual republish.
11. **No global exactly-once assumption:** Kafka transactions do not automatically make Kafka, SQL databases, and arbitrary external side effects globally atomic.

---

## 5. Requirements clarification tracker

The following questions should be resolved before the design is considered final.

| Topic | Current assumption | Clarification needed |
|---|---|---|
| 10 ms latency | Distribution-path target | What are the exact start/end timestamps? Is the requirement p99, p99.9, or a hard maximum? Does it include destination broker acknowledgement? |
| Ordering | Key by `QuoteID` | Is ordering required per quote, RFQ, dealer, or some combination? Must each destination observe strict order? |
| Throughput | Horizontally scalable Kafka-based system | What are average and peak events/second, payload size, and burst duration? |
| 60-day retention | Durable archive plus Kafka retention | Is the archive required to be immutable? What recovery/query patterns and RPO/RTO apply? |
| Delivery semantics | At-least-once with idempotent consumers | Are duplicate messages acceptable if duplicate business effects are prevented? |
| Destination availability | Retry asynchronously | What are per-destination recovery objectives and maximum acceptable delivery delay? |
| Replay scope | Manual replay for outages/DR/rollback | Replay an event range, partition/time range, event ID, destination, or full dataset? |
| Security | Controlled access and audited replay | What authentication, authorization, encryption, and data-classification controls are required? |
| Invalid events | Durable invalid-event record | How long must invalid events be retained, and who owns their correction/reprocessing? |
| Archive failure | Independent History Writer | What maximum archive lag is acceptable, and how do we recover before Kafka source records expire? |

---

## 6. Architecture implications of the requirements

### 6.1 Ingestion durability

The ingestion response should reflect the agreed durability boundary. If the service acknowledges before the event is durable, a failure can lose an event that the producer believes was accepted.

### 6.2 Separate live distribution and history archival

The Distributor and History Writer consume the same source topic using separate consumer groups. This isolates their consumer progress, but it does not eliminate the need to monitor the History Writer's lag. If source records expire before archival catches up, the archive can have gaps.

### 6.3 Retry and offset safety

Do not commit a source offset while failed delivery work exists only in memory. Required delivery work must be completed or durably handed off to a recoverable mechanism first. Since a database write and Kafka offset commit are not automatically atomic, retry-task insertion and state transitions must be idempotent.

### 6.4 Idempotency boundary

Use stable EventIDs. For a destination that updates a database, store the processed EventID and apply the business update in the same transaction where possible. External side effects need their own idempotency support, outbox/workflow, or reconciliation.

### 6.5 60-day recovery

Kafka retention can support short-term replay and consumer catch-up. A separate durable archive supports historical recovery when source-topic retention is insufficient or Kafka is unavailable. The archive writer must be monitored and have its own recovery plan.

---

## 7. Summary for a system-design interview

“I would build a Kafka-based event distribution pipeline. The ingestion service validates events and acknowledges only after the selected durable Kafka write succeeds. A Distributor consumer group routes valid events to downstream topics, while a separate History Writer consumer group archives all valid events for 60 days. I would use a stable EventID for idempotency, a confirmed partition key for required ordering, and durable retry/recovery state so no source offset is committed while failed work exists only in memory. Persistent failures go to an auditable DLQ/recovery workflow, and manual replay preserves the original EventID. I would monitor distribution latency, consumer lag, archive lag, retries, and DLQ age. Before finalizing the design, I would clarify whether 10 ms is p99 or a hard maximum, the exact latency boundary, required ordering scope, peak throughput, and the archive's RPO/RTO.”

---

## 8. Requirement checklist

Use this checklist to track review progress; checking an item means it has been reviewed, not that the implementation is complete.

- [ ] FR-01 — Ingestion
- [ ] FR-02 — Validation and durable invalid-event handling
- [ ] FR-03 — Destination routing
- [ ] FR-04 — Ordering scope
- [ ] FR-05 — 60-day archive
- [ ] FR-06 — Manual replay
- [ ] FR-07 — Retry and failed-delivery recovery
- [ ] FR-08 — Idempotency
- [ ] FR-09 — No silent loss
- [ ] FR-10 — Traceability
- [ ] NFR-01 — 10 ms latency definition
- [ ] NFR-02 — Durability and reliability
- [ ] NFR-03 — 60-day retention
- [ ] NFR-04 — Scalability
- [ ] NFR-05 — Fault tolerance
- [ ] NFR-06 — Ordering guarantees
- [ ] NFR-07 — Duplicate-safe processing
- [ ] NFR-08 — Recoverability
- [ ] NFR-09 — Observability
- [ ] NFR-10 — Security and auditability

---

**End of requirements document.**
