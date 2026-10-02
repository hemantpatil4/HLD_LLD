# Bloomberg URL Shortener — Capacity Design Deep Dive

## 1. What is capacity design?

Capacity design means estimating:

- How much traffic the system receives
- How much data it stores
- How much memory/cache is required
- How much database capacity is required
- How many API instances are needed
- What happens during traffic spikes

The goal is not perfect prediction.

The goal is:

> Make reasonable assumptions and verify that the architecture can handle the expected scale.

---

## 2. Start with traffic assumptions

Suppose:

```text
New URLs created = 10,000/sec
Redirect requests = 1,000,000/sec
```

This is a read-heavy system.

Ratio:

```text
1,000,000 / 10,000 = 100
```

Approximately:

```text
1 write : 100 reads
```

Therefore caching becomes extremely important.

---

## 3. Daily traffic

Redirects:

```text
1,000,000 requests/sec
```

Per day:

```text
1,000,000 × 86,400
= 86.4 billion redirects/day
```

Writes:

```text
10,000 × 86,400
= 864 million URLs/day
```

These numbers are intentionally illustrative.

In an interview, clearly state:

> "I'll use these as capacity assumptions; the exact numbers can be adjusted."

---

## 4. Average vs peak traffic

Average traffic is not enough.

Suppose:

```text
Average = 1M requests/sec
Peak multiplier = 3x
```

Then design target:

```text
Peak = 3M requests/sec
```

Architecture:

```text
Average
   |
   v
1M RPS

Peak
   |
   v
3M RPS
```

You need headroom.

---

## 5. API server capacity

Suppose one API instance can safely handle:

```text
100,000 requests/sec
```

Peak traffic:

```text
3,000,000 requests/sec
```

Minimum instances:

```text
3,000,000 / 100,000
= 30
```

But running exactly 30 is risky.

Add headroom:

```text
30 × 1.5
= 45 instances
```

So perhaps:

```text
45 API instances
```

The exact number depends on real benchmarks.

---

## 6. Why benchmark matters

Do not say:

> "One server handles 100K RPS."

without testing.

Actual capacity depends on:

- CPU
- Memory
- Network
- Request complexity
- Serialization
- Redis latency
- Database calls
- Logging
- TLS
- Runtime behavior

A better interview answer:

> "I would benchmark a representative API instance under realistic workload, then size instances using peak traffic plus headroom."

---

## 7. Redis capacity

Suppose:

```text
1M reads/sec
```

and cache hit ratio is:

```text
99%
```

Redis handles approximately:

```text
990,000 requests/sec
```

Database sees approximately:

```text
10,000 requests/sec
```

This is a massive difference.

```text
1M requests
      |
      v
Redis
 |         \
99% HIT     1% MISS
 |            |
302          DB
```

This is why cache hit ratio is a critical capacity metric.

---

## 8. Cache hit ratio

Formula:

```text
Hit Ratio =
Cache Hits / Total Cache Requests
```

Example:

```text
990,000 hits
10,000 misses
----------------
1,000,000 total

Hit ratio = 99%
```

Miss ratio:

```text
1%
```

If hit ratio drops from 99% to 80%:

```text
DB load increases dramatically.
```

Therefore:

> Cache capacity and cache effectiveness directly affect database capacity.

---

## 9. Database storage estimation

Suppose each URL record averages:

```text
Original URL       = 300 bytes
ID                 = 8 bytes
Metadata/indexes   = additional overhead
```

Use a rough average:

```text
500 bytes/record
```

At:

```text
864M new URLs/day
```

Raw data:

```text
864M × 500 bytes
≈ 432 GB/day
```

One year:

```text
432 GB × 365
≈ 158 TB
```

This is before considering replication, indexes, backups and storage overhead.

The point of the calculation is to discover whether a single database is realistic.

---

## 10. Database IOPS

Suppose cache miss rate is:

```text
1%
```

and redirects are:

```text
1M/sec
```

Database reads:

```text
1M × 1%
= 10K reads/sec
```

Writes:

```text
10K new URLs/sec
```

So the database might see approximately:

```text
10K reads/sec
10K writes/sec
```

Again, these are simplified estimates.

---

## 11. Network bandwidth

Suppose each redirect response/request path consumes roughly:

```text
2 KB
```

At:

```text
1M requests/sec
```

Approximate traffic:

```text
1M × 2 KB
= 2 GB/sec
```

This is why network capacity must also be considered.

Real traffic depends on:

- Request headers
- Response headers
- TLS
- HTTP version
- Redirect behavior
- CDN
- Compression where applicable

---

## 12. Connection pool sizing

Suppose:

```text
100 API instances
```

and each instance allows:

```text
100 DB connections
```

Potential maximum:

```text
100 × 100
= 10,000 DB connections
```

That may overwhelm the database.

Therefore:

> Scaling API instances also changes downstream connection pressure.

Use connection pooling and size pools based on actual DB capacity.

---

## 13. Cache memory estimation

Suppose we want to cache:

```text
100 million hot URLs
```

Average cached object:

```text
1 KB
```

Raw memory:

```text
100M × 1 KB
≈ 100 GB
```

Redis also needs overhead for:

- Keys
- Values
- Metadata
- Expiration
- Data structures
- Replicas

Therefore actual memory required is larger.

---

## 14. Headroom

Never design exactly at the expected peak.

Example:

```text
Expected peak = 1M RPS
Target capacity = 1.5M RPS
```

Headroom:

```text
50%
```

Why?

- Traffic spikes
- Deployment
- Instance failures
- Cache misses
- Uneven traffic
- Operational events

---

## 15. Capacity planning loop

```text
Traffic assumptions
        |
        v
Capacity calculation
        |
        v
Architecture
        |
        v
Benchmark
        |
        v
Observe production
        |
        v
Update assumptions
        |
        +------> repeat
```

---

## 16. URL Shortener capacity architecture

```text
                         Peak Traffic
                             |
                             v
                       Load Balancer
                             |
                +------------+------------+
                |            |            |
               API          API          API
                |            |            |
                +------------+------------+
                             |
                       Redis Cluster
                       /     |      \
                    Shard   Shard   Shard
                             |
                         Cache Miss
                             |
                             v
                       Read Replicas
                             |
                             v
                         Primary DB
```

---

## 17. Interview answer

If asked "How do you capacity-plan this?":

> "I start by estimating write and read QPS, peak multiplier, data growth and cache hit ratio. Because URL shorteners are read-heavy, I expect Redis to absorb most redirect traffic. I then benchmark API instances, Redis and database nodes, size them for peak traffic with headroom, and account for downstream limits such as DB connections and storage growth. Finally, I validate the assumptions with production metrics and load tests."

---

## 18. Key formulas

```text
Daily Requests
= RPS × 86,400

Required Instances
= Peak RPS / Safe RPS per instance

Cache Hit Ratio
= Hits / Total Requests

DB Miss Traffic
= Total Traffic × Cache Miss %

Storage
= Records × Average Record Size

Headroom Capacity
= Expected Peak × Safety Factor
```

---

## 19. Key takeaway

Capacity design is not just:

```text
"How many servers?"
```

It is:

```text
Traffic
  +
Data
  +
Memory
  +
CPU
  +
Network
  +
Connections
  +
Peak
  +
Failure headroom
```
