# Bloomberg URL Shortener — Database Scaling & Sharding Deep Dive

## 1. Start simple
```text
API -> Database
```
Before sharding, fix fundamentals:
- correct schema
- indexes
- efficient queries
- connection pooling
- sensible transactions

## 2. Indexing
Redirect lookup:
```sql
SELECT Id, OriginalUrl
FROM UrlMappings
WHERE Id = 125;
```
`Id` should normally be the primary key/indexed column.

EF Core:
```csharp
var mapping = await db.UrlMappings.FindAsync(id);
```

## 3. Connection pooling
Do not open a brand-new physical connection for every request:
```text
Requests -> Connection Pool -> DB
```
Too many API instances * pool size can overwhelm the DB.

## 4. Batching
For bulk workloads:
```text
100 individual inserts -> many round trips
1 batch -> fewer round trips
```
Interactive requests still need appropriate latency semantics.

## 5. Vertical scaling
```text
16 CPU / 64 GB
      |
      v
64 CPU / 256 GB
```
Simple, but has hardware/cost limits.

## 6. Read replicas
```text
             Primary
             /     \
            v       v
           R1       R2
```
Writes -> primary.
Reads -> replicas.
Useful for read scaling, not primary write scaling.

## 7. Replication lag
Async replication:
```text
Primary -> later -> Replica
```
A just-created URL may not yet exist on a replica.

Mitigations:
- populate Redis on write
- route consistency-sensitive reads to primary
- read-your-write routing
- stronger replication if required

## 8. Partitioning vs sharding
Partitioning:
```text
One DB system
  |
  +-- partition A
  +-- partition B
```

Sharding:
```text
Application
 /   |   \
DB1 DB2 DB3
```

Shards are separate data owners.

## 9. Why shard?
If one DB cannot handle:
- write throughput
- storage size
- aggregate workload

split the dataset:
```text
300k writes/sec
   |
   +-> DB1 100k
   +-> DB2 100k
   +-> DB3 100k
```

## 10. Range sharding
```text
1..1M       -> DB1
1M..2M      -> DB2
2M..3M      -> DB3
```
Simple and good for range queries.

Problem for sequential IDs:
```text
new writes -> newest range -> newest shard becomes hot
```

## 11. Hash sharding
Concept:
```text
hash(id) -> shard
```
Example:
```text
ID -> stable hash -> shard number
```
Good for distributing point lookups/writes.

A simplistic modulo example:
```csharp
int shard = (int)(id % shardCount);
```
This is easy to understand but changing `shardCount` remaps many keys.

## 12. Consistent hashing
A hash ring reduces the amount of data that must move when nodes change:
```text
             Shard A
                |
        +-------+-------+
        |               |
     Shard D          Shard B
        |               |
        +-------+-------+
                |
             Shard C
```

## 13. Virtual nodes
A physical shard owns multiple logical positions on the ring:
```text
DB1 -> vnode1, vnode4, vnode7
DB2 -> vnode2, vnode5
DB3 -> vnode3, vnode6
```
This improves distribution and rebalancing.

## 14. Shard key
For the URL shortener, the dominant access pattern is:
```text
short code -> exact ID -> one mapping
```
A conceptual strategy:
```text
Snowflake ID
    |
    v
stable hash
    |
    v
shard
```

## 15. Cross-shard queries
If a query does not include the shard key:
```text
Query
 / | \
v  v  v
DB1 DB2 DB3
 \  |  /
  merge
```
This scatter-gather pattern is expensive.

## 16. Cross-shard transactions
```text
Transaction
  +-> DB1
  +-> DB2
```
Distributed transactions add latency and complexity. Prefer operations that stay inside one shard.

## 17. Shard replicas
```text
Shard 1: Primary + Replica
Shard 2: Primary + Replica
Shard 3: Primary + Replica
```
Sharding provides scale.
Replication provides redundancy/availability.

## 18. Hot shards
Even a good scheme can become imbalanced:
```text
DB1 10%
DB2 10%
DB3 70%
DB4 10%
```
Causes:
- poor shard key
- hot tenants
- hot keys
- traffic skew

Mitigations:
- better distribution
- virtual nodes
- split hot partitions
- cache hot values
- replicas/read scaling

## 19. Connection-pool math
If:
```text
100 API instances
100 DB connections each
```
the DB may see up to:
```text
10,000 connections
```
So horizontal API scaling must be coordinated with DB capacity.

## 20. Backpressure
When DB is overloaded:
```text
Requests -> DB -> overload
```
Do not retry indefinitely.
Use:
- timeouts
- bounded concurrency
- exponential backoff
- circuit breaker
- rate limiting
- queues for asynchronous workloads

## 21. Time partitioning
If URLs expire, time partitions can help:
```text
2026-01
2026-02
2026-03
```
Useful for lifecycle management, but the current partition can become write-hot.

## 22. Database observability
Track:
```text
CPU
Memory
IOPS
Query latency
Connection count
Pool usage
Lock waits
Deadlocks
Replication lag
Read/write throughput
Shard utilization
Hot partitions
```

## 23. Recommended scaling progression
Do not start with sharding:
```text
Schema
  -> indexes
  -> query tuning
  -> connection pooling
  -> vertical scaling
  -> read replicas
  -> partitioning
  -> sharding
```
The exact order depends on workload.

## 24. Interview answer
> I would not start with sharding. First I would optimize indexes and queries, use connection pooling, scale vertically where appropriate, and add read replicas for read traffic. If the primary's write throughput, storage capacity, or dataset size becomes the bottleneck, I would shard. For this URL shortener, because the dominant operation is a point lookup by ID, a stable hash of the ID can distribute mappings across shards while keeping each redirect on one shard.
