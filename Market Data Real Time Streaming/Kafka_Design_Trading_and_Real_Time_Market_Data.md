# Kafka Design for Trading Events and Real-Time FX Market Data
## Core concepts using our same example, topic names, and partition/offset discussion

## 1. What Kafka contributes

Kafka is a distributed event log. It stores records in topic partitions, allows multiple consumer groups to read independently, absorbs bursts, and lets consumers resume/replay while data is retained.

In our design, Kafka is useful for:
- Durable buffering between ingestion and distribution.
- Parallel processing through partitions.
- Independent consumers such as Distributor and History Writer.
- Replication and recovery from broker/node failures.
- Replay within configured retention.

Kafka does **not** automatically guarantee:
- End-to-end 10 ms latency.
- Exactly-once business effects across arbitrary services/databases.
- 60-day retention unless configured and capacity-planned.
- Atomic success across multiple destination topics.
- Parallel processing of one hot key while preserving strict per-key order.

## 2. Architecture

```text
Trading Event Producer
        |
        v
 Event Ingestion Service
        | valid                 | invalid
        v                       v
 trade-events             invalid-trade-events
        |
        +----------------------------+
        |                            |
        v                            v
 Trade Distributor              History Writer
 group: trade-distributor-group group: trade-history-writer
        |                            |
        v                            v
 risk-events / reporting-events   60-day archive
 settlement-events / audit-events
```

The History Writer is in a different consumer group, so it reads the source stream independently and has its own offsets.

## 3. Cluster, broker, topic, partition, record

- **Cluster:** all Kafka brokers working together.
- **Broker:** one Kafka server.
- **Topic:** named stream, e.g. `trade-events`.
- **Partition:** ordered log inside a topic; enables parallelism.
- **Record:** one event with a key, payload, timestamp, and possibly headers.

```text
Kafka cluster
  Broker 1       Broker 2       Broker 3
      \             |             /
       +------ topic partitions --+
                    |
               trade-events
              /    |     \
            P0     P1     P17 ... P63
```

A partition has a leader replica at a time and may have replicas on other brokers. Producers write through the partition leader. Replication helps durability and availability when configured appropriately.

## 4. Topics in our example

| Topic | Role |
|---|---|
| `trade-events` | Original valid trading event stream. |
| `risk-events` | Events routed to Risk. |
| `reporting-events` | Events routed to Reporting. |
| `settlement-events` | Events routed to Settlement. |
| `audit-events` | Events routed to Audit. |
| `invalid-trade-events` | Invalid events plus reason metadata. |

One source event can be sent to several destination topics: this is fan-out. `QUOTE_HIT` might go to Risk, Reporting, and Audit.

## 5. The same event and Kafka location

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

For our example:
```text
Topic: trade-events
Key: QUOTE-9001
Partition: P17
Offset: 102
EventID: EVT-1003
```

- `EventID` identifies one event.
- `QuoteID` groups events in one quote lifecycle and can be the Kafka key.
- `sequenceNumber` is a business sequence.
- Kafka offset is the record's position in a particular partition.
These are not interchangeable.

## 6. Keys and partitions

We use `QuoteID` as the key when ordering by quote is required. Kafka's partitioner maps that key to a partition. Events with the same key normally map to the same partition under a stable partitioning setup.

```text
trade-events (64 illustrative partitions)
 P0: many quote keys
 P1: many quote keys
 ...
 P17: EVT-1003 at offset 102
 ...
 P63: many quote keys
```

64 partitions and replication factor 3 are working examples, not final capacity decisions.

Kafka guarantees order within one partition, not globally across every partition. Different quote keys can be processed in parallel. A hot quote stays on one partition if strict per-quote ordering is required; adding partitions spreads other keys but does not split that one key.

Do not assume a particular key maps to P17 without the actual partitioner, key bytes, partition count, and configuration.

## 7. Partition sizing and capacity

Estimate partition count by measuring:
1. Peak events/second and burst size.
2. Average and maximum record size.
3. Throughput per partition and consumer.
4. Number of consumers and desired parallelism.
5. Broker CPU, disk, network, replication, and retention.
6. Key skew/hot partitions.
7. Latency under peak load, failover, and replay.

More partitions are not automatically better. They add overhead, and changing partition count can change the mapping of keys for future records.

## 8. Replication and producer acknowledgement

Replication factor 3 means Kafka maintains three replicas per partition, normally spread across brokers.

A common durability setup considers:
- `acks=all`
- Appropriate replication factor.
- Appropriate `min.insync.replicas`.
- Producer idempotence where supported/configured.
- Appropriate retry and delivery timeout values.

`acks=all` waits for acknowledgement from the in-sync replica set under the cluster configuration. It does not mean Risk or Reporting completed its business processing.

The ingestion API should return success only after the Kafka publish has met the durability condition selected for the design.

## 9. Consumer groups and rebalances

The Distributor pods share:
```text
GroupId = trade-distributor-group
```

Kafka assigns partitions among members of that group. One partition is assigned to at most one active consumer in the group at a time; one consumer can own multiple partitions.

The History Writer uses:
```text
GroupId = trade-history-writer
```

It independently consumes the same source topic. A different group has independent committed offsets.

When a consumer crashes or group membership changes, Kafka rebalances partitions. A rebalance may briefly pause processing depending on client/assignment settings. If there are more consumers than partitions, some consumers are idle.

## 10. Offsets and crash recovery

For partition P17:
```text
offset 100 -> event A
offset 101 -> event B
offset 102 -> EVT-1003
offset 103 -> event D
```

The consumer application decides when to commit; Kafka stores the committed position. In the common consumer API convention, the committed value is the next offset to read.

- Commit 102 -> resume at 102.
- Commit 103 -> resume at 103.

### Not completed
Pod A reads offset 102 and crashes before completing/committing. Pod B takes P17 after rebalance and resumes at 102.

### Completed
Pod A completes required work and commits 103, then crashes. Pod B resumes at 103.

Fetching is not processing completion. Commit only when required outcomes are safely handled.

## 11. Partial destination failure

```text
trade-events / P17 / offset 102
                 |
                 v
            Distributor
              /      \
             v        v
       risk-events  reporting-events
         success       failure
```

The destination publishes and input offset commit are separate operations. If the pod crashes before committing 103, the event can be retried and Risk may receive a duplicate.

Possible designs:
- Retry all destinations and use downstream idempotency.
- Keep durable per-destination delivery status and retry only pending destinations.
- Use Kafka transactions for suitable Kafka-to-Kafka consume-transform-produce flows.

Even with delivery status, a crash can happen after Risk accepts the message but before `SUCCESS` is recorded. Therefore downstream idempotency remains important.

## 12. Idempotency with EventID

Downstream consumers can persist processed IDs and ensure the business effect and processed-ID record are atomic, commonly in one database transaction.

Illustrative pseudocode:
```csharp
await using var tx = await db.BeginTransactionAsync();

if (await repository.IsProcessedAsync(event.EventId, tx))
{
    await tx.CommitAsync();
    return;
}

await ApplyBusinessEffectAsync(event, tx);
await repository.MarkProcessedAsync(event.EventId, tx);
await tx.CommitAsync();
```

This is conceptual code; actual APIs depend on the data layer. `EventID` must be generated once and reused across retries. It does not prevent duplicate transport; it prevents duplicate business effects when implemented correctly.

Kafka transactions can coordinate supported Kafka input/output/offset operations. They do not automatically make a database update and an external service call globally atomic.

## 13. Retries, DLQ, and invalid events

- Retry transient failures with bounded retries and backoff.
- Invalid payloads should be recorded with the reason in `invalid-trade-events`.
- Repeatedly failing delivery needs a durable retry/DLQ process, monitoring, retention, and operator ownership.
- Do not advance source progress until the event's required outcomes are handled or the event is durably handed to an explicitly designed recovery workflow.

A DLQ does not itself solve recovery; it needs a replay/correction process.

## 14. 60-day history and manual replay

```text
trade-events
   |------------------|
   v                  v
Distributor       History Writer
                      |
                      v
                 60-day archive
                      |
                      v
              controlled replay tool
                      |
                      v
             selected destination topic
```

Use a separate History Writer consumer group. Kafka's own retention must cover the expected outage/recovery window. The archive is the long-term 60-day history store in our initial design.

Replay should be deliberate and audited. Preserve the original `EventID` when replaying the same business event, and make consumers idempotent.

## 15. Real-time FX market-data design

The same Kafka principles apply to market data, but the semantics are different.

```text
Bloomberg / Reuters feeds
          |
          v
+--------------------------+
| Feed Handler             |
| parse/normalize/validate |
| track source sequence    |
| detect sequence gaps     |
+--------------------------+
          |
          v
      market-events
          |
          +------------------+
          |                  |
          v                  v
   Pricing / Risk       History Writer
          |
          v
 Latest-price cache / subscription routing
          |
          v
 WebSocket Gateway fleet
          |
          v
      Client apps
```

A normalized FX update could be:

```json
{
  "eventId": "MD-EURUSD-1001",
  "instrument": "EUR/USD",
  "source": "Reuters",
  "bid": 1.08450,
  "ask": 1.08462,
  "sourceSequence": 1001,
  "eventType": "FX_QUOTE_UPDATE"
}
```

This is a separate market-data event, not the trade event `EVT-1003`. It uses the same concepts:
- `eventId` identifies this normalized update.
- `instrument` can be the Kafka key when per-instrument ordering matters.
- `sourceSequence` detects gaps in the upstream feed.
- Kafka offset identifies the record's position in a Kafka partition.

### Market-data partitions

A possible topic is `market-events`, with 64 illustrative partitions:

```text
market-events (64 illustrative partitions)
 P0  -> some instrument keys
 P1  -> some instrument keys
 ...
 P17 -> some instrument keys
 ...
 P63 -> some instrument keys
```

Use `instrument` as the key if per-instrument order is required. Do not claim EUR/USD maps specifically to P17 without actual partitioner details. P17/offset 102 is the fixed placement in our trade-event example only.

Possible market-data topics include `market-events`, `pricing-events`, `risk-market-events`, and `market-audit-events`. These are illustrative; do not create a topic for every consumer unless isolation or routing requirements justify it.

Do not send market-data records into `risk-events` or `reporting-events` from the trade example unless the business contract explicitly defines that. Keep trade-event and market-data contracts distinct.

## 16. Trade events versus market data

| Concern | Trade event | Market-data event |
|---|---|---|
| Example | `EVT-1003`, `QUOTE_HIT` | `MD-EURUSD-1001`, `FX_QUOTE_UPDATE` |
| Ordering key | `QuoteID` | `instrument` |
| Business/source sequence | `sequenceNumber = 3` | `sourceSequence = 1001` |
| Kafka position | `trade-events`, P17, offset 102 | `market-events`, partition based on key/config |
| Main concern | Correct routing, durable processing, business effect | Freshness, sequence gaps, low latency, subscriber fan-out |
| Slow consumer | Usually cannot silently lose business events | May coalesce updates only when requirements allow |

## 17. WebSocket gateways and slow clients

For market data:
1. Feed Handler normalizes a tick.
2. The stream/cache updates the latest quote.
3. Subscription routing finds clients interested in the instrument.
4. A WebSocket gateway sends the update.
5. A slow client must not block other clients.

Possible policies:
- Bounded per-client queues.
- Coalesce pending updates to the latest price when permitted.
- Disconnect or drop for persistently slow clients according to an explicit policy.
- Keep durable history separate from best-effort live delivery.
- Monitor queue depth, event age, publish-to-client latency, dropped/coalesced updates, and disconnects.

Do not blindly coalesce trade events; often every trade event must be preserved.

## 18. Observability

Monitor:
- Produce success/failure and latency.
- Consumer lag by group and partition.
- Rebalances/restarts.
- Source-to-destination processing latency.
- Destination publish failures/retries.
- DLQ volume and oldest-message age.
- Hot partition/key distribution.
- Broker CPU, disk, network, replica health.
- History Writer lag and archive failures.
- End-to-end event age and market-data freshness.

Lag alone is not enough. A system can have low lag and still have stale events or high end-to-end latency.

## 19. Questions to resolve before finalizing HLD

1. Is 10 ms a hard max, p99, or p99.9?
2. Does timing include broker acknowledgements for all destination topics?
3. What are peak events/second, burst size, and payload size?
4. How many destinations receive a typical event?
5. How much Kafka retention is needed to cover outages?
6. How much data is required for 60 days and what archive storage is used?
7. What is the hot-key strategy?
8. What is the retry/backoff/DLQ policy?
9. What delivery guarantee is required, and across which boundary?
10. How will manual replay be audited and deduplicated?

## 20. Interview-ready summary

“Kafka provides a durable, partitioned event log and allows separate consumer groups to read the same stream independently. I would key trading events by `QuoteID` for quote-level ordering and use `EventID` for deduplication. The Distributor evaluates routing rules and publishes to destination topics, committing its source offset only after required outcomes are safely handled. Since partial publishing can lead to duplicates, I would use idempotent downstream processing and durable delivery tracking where needed. A separate History Writer stores events for 60 days. For market data, I would use `instrument` as the key when per-instrument ordering matters, track source feed sequence gaps, protect WebSocket gateways from slow clients, and validate the full latency budget with load tests.”

## Quick glossary

| Term | Meaning |
|---|---|
| Topic | Named stream of records. |
| Partition | Ordered log within a topic. |
| Key | Helps choose partition and preserve per-key ordering. |
| Offset | Record position in one partition. |
| Consumer group | Consumers sharing partitions for one job. |
| Rebalance | Reassignment after group membership changes. |
| Replication factor | Number of replicas for a partition. |
| `acks=all` | Wait for configured in-sync replica acknowledgements. |
| Consumer lag | How far a group is behind latest records. |
| Idempotency | Repeating an operation does not duplicate its business effect. |
| DLQ | Durable location for invalid/repeatedly failing events. |
| Replay | Process selected stored events again. |
| Hot key | One key with disproportionately high traffic. |
| Source sequence | Upstream market-data sequence, distinct from Kafka offset. |
| Coalescing | Replacing several pending updates with the latest, if allowed. |
