# Bloomberg URL Shortener — Consistency & CAP Deep Dive

## 1. Why does consistency matter?

Our system has multiple copies/components:

```text
API
 |
Redis
 |
Primary DB
 |
Read Replicas
```

If data is copied between machines, those copies may temporarily disagree.

Example:

```text
Primary DB:
ID 125 -> google.com

Replica:
ID 125 -> not present yet
```

This can happen with asynchronous replication.

---

## 2. What is consistency?

At a simple level:

> Consistency describes what value a read is allowed to observe relative to writes.

Example:

```text
Write:
ID 125 -> google.com
```

Then:

```text
Read ID 125
```

A strong consistency expectation might be:

```text
Read -> google.com
```

A weaker model may temporarily allow:

```text
Read -> not found
```

if the read is served from a lagging replica.

---

## 3. Strong consistency

Conceptually:

```text
WRITE
  |
  v
System confirms write
  |
  v
Subsequent READ
  |
  v
Latest value
```

Useful when clients must immediately observe the latest state.

Trade-off:

- More coordination
- Potentially higher latency
- Lower availability during some failures/partitions

---

## 4. Eventual consistency

With asynchronous replication:

```text
Primary
   |
   | async replication
   v
Replica
```

For some period:

```text
Primary = new value
Replica = old value
```

Eventually:

```text
Replica = new value
```

This is eventual consistency.

---

## 5. URL shortener example

Create:

```text
POST /api/urls
```

Primary generates:

```text
ID = 125
Code = 21
URL = https://google.com
```

Then:

```text
Primary DB -> contains 125
Replica -> replication still pending
```

Immediately:

```text
GET /21
```

If routed to replica:

```text
Replica -> not found
```

Possible response:

```text
404
```

even though creation succeeded.

This is the classic read-after-write issue.

---

## 6. Read-after-write consistency

A client expects:

```text
WRITE X
   |
   v
READ X
   |
   v
must see X
```

Possible solutions:

### Read from primary

After a write, route the relevant read to the primary.

### Cache on write

When creating:

```text
Primary DB
   |
   v
Redis
```

Then redirect can read Redis immediately.

### Session-aware routing

Remember that a client recently wrote data and route its reads appropriately.

---

# CAP THEOREM

## 7. What is CAP?

CAP describes a fundamental trade-off in a distributed system during a network partition.

CAP stands for:

```text
C = Consistency
A = Availability
P = Partition Tolerance
```

---

## 8. C — Consistency

For CAP discussion, consistency means that every read receives the most recent write (or an error), according to the system's consistency definition.

Example:

```text
Write:
x = 10

Read:
x -> 10
```

No stale copy is returned.

---

## 9. A — Availability

Availability means every request to a non-failed node receives a non-error response.

Simplified:

```text
Request
  |
  v
Healthy distributed system
  |
  v
gets a response
```

The response may not necessarily be the newest value in an AP-style system.

---

## 10. P — Partition Tolerance

A network partition means distributed nodes cannot communicate reliably.

Example:

```text
Node A
  |
  X  NETWORK PARTITION
  |
Node B
```

Both nodes may still be running.

The problem is:

```text
A cannot reliably communicate with B.
```

---

## 11. Why is P important?

In a distributed system, network failures can happen.

You cannot simply assume:

```text
Network = always reliable
```

Therefore real distributed systems need a partition strategy.

The important CAP question is:

> What does the system do when communication between nodes breaks?

---

## 12. The CAP trade-off

During a partition:

```text
Node A <---- X ----> Node B
```

Suppose both receive a request to update the same value.

To preserve strict consistency, the system may need to stop or reject some operations until coordination is restored.

That sacrifices availability.

Alternatively, the system can continue accepting requests independently.

That can sacrifice strong consistency.

Therefore:

```text
During partition:

CP
Consistency + Partition Tolerance
        vs
AP
Availability + Partition Tolerance
```

The simplified phrase "pick two of three" is useful for interviews, but the more precise idea is:

> When a partition occurs, a distributed system cannot simultaneously guarantee strong consistency and availability for all operations.

---

## 13. CP example

Suppose two database nodes cannot communicate.

For a strongly consistent operation:

```text
Client
  |
Node A
  |
  X
Node B
```

If Node A cannot confirm the required state with Node B, it may reject/wait rather than return a potentially inconsistent result.

Result:

```text
Consistency preserved
Availability reduced
Partition tolerated
```

---

## 14. AP example

An AP-oriented system may continue serving requests independently during a partition.

```text
Client -> Node A -> success
Client -> Node B -> success
```

But the nodes may temporarily have different state.

Later they reconcile.

Result:

```text
Availability preserved
Partition tolerated
Strong consistency sacrificed
```

---

## 15. Important: CAP is not "database vs cache"

A common mistake is:

> "Redis is AP and SQL is CP."

That is too simplistic.

CAP depends on:

- Specific distributed system
- Replication model
- Failure scenario
- Consistency guarantees
- Implementation/configuration

Do not assign a universal CAP label to a technology without context.

---

## 16. Replication lag is not automatically CAP

Suppose:

```text
Primary
  |
  | 100 ms delay
  v
Replica
```

The replica is temporarily stale.

That is replication lag.

It does not automatically mean a CAP partition occurred.

CAP becomes relevant when there is a distributed communication/failure partition and we must choose how the system behaves under that partition.

---

## 17. URL shortener and CAP

A useful architecture:

```text
              API
               |
        +------+------+
        |             |
      Redis         DB
                      |
                +-----+-----+
                |           |
             Primary      Replica
```

For URL creation:

```text
POST
 |
Primary
 |
commit
 |
Redis
 |
response
```

For redirect:

```text
GET
 |
Redis HIT
 |
302
```

This reduces the need to consult replicas for hot URLs.

---

## 18. Why URL shortener can tolerate some eventual consistency

Suppose URL mappings are immutable:

```text
aZ91k -> https://google.com
```

Once created, the mapping usually does not change.

That makes eventual consistency easier to tolerate in some parts of the system.

However, creation must still provide a reliable user experience.

A good strategy is:

```text
Write Primary
     |
     +----> Populate Redis
     |
     v
Response
```

Then the redirect path can immediately find the mapping in Redis.

---

## 19. Consistency choices by operation

### URL creation

Prefer:

```text
Strong confirmation of write
```

because the user expects creation to succeed before receiving the short URL.

### Redirect

Can often rely on:

```text
Redis
```

after the mapping has been written.

### Analytics

Clicks/views may be processed asynchronously:

```text
Redirect
   |
   +--> immediate 302
   |
   +--> event queue
            |
            v
        analytics
```

Analytics often does not need strict synchronous consistency.

---

## 20. Interview answer — CAP

> "CAP becomes relevant when distributed nodes experience a network partition. At that point, a system cannot guarantee both strong consistency and availability for all operations while tolerating the partition. For our URL shortener, I would keep the source-of-truth write path strongly controlled through the primary database and use Redis to serve the high-volume redirect path. I would explicitly handle replica lag and read-after-write requirements rather than treating every replication delay as a CAP problem."

---

## 21. Interview answer — consistency

> "I would separate consistency requirements by operation. URL creation needs a confirmed durable write before returning the short URL. Redirects can primarily use Redis, which avoids replica-lag problems for hot mappings. Analytics can be eventually consistent because it doesn't need to block the redirect response."

---

## 22. Key takeaway

```text
Consistency
    |
    +--> What value can a read observe?

CAP
    |
    +--> C = Consistency
    +--> A = Availability
    +--> P = Partition Tolerance

During partition:
    Strong consistency <-> Availability trade-off
```

Remember:

> Replication lag is a consistency concern; a network partition is the failure condition that makes the CAP trade-off relevant.
