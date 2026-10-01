# Bloomberg URL Shortener — Complete System Design Revision

## 1. Requirements

### Functional
- Create a short URL.
- Redirect a short URL.
- Store the original URL.
- Generate a unique identifier.

### Non-functional
- Low redirect latency.
- High read throughput.
- High availability.
- Horizontal scalability.
- Durable mappings.
- Controlled failure behavior.
- Security and abuse prevention.
- Observability.

---

## 2. APIs

Create:
```http
POST /api/urls
Content-Type: application/json

{
  "url": "https://example.com/products/123"
}
```

Response:
```json
{
  "shortUrl": "https://short.ly/aZ91k"
}
```

Redirect:
```http
GET /aZ91k
```

Response:
```http
302 Found
Location: https://example.com/products/123
```

---

## 3. Start with the simplest architecture

```text
Client
  |
  v
ASP.NET Core API
  |
  v
SQL Server
```

Data:
```text
ID       OriginalUrl
------------------------------
125      https://google.com
126      https://amazon.com
```

Create:
```text
POST -> INSERT -> generated ID -> Base62 -> short URL
```

Redirect:
```text
GET /21 -> Base62 decode -> ID 125 -> DB -> URL -> 302
```

---

## 4. Base62

Alphabet:
```text
0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
```

Example:
```text
125 / 62 = 2 remainder 1
2 / 62   = 0 remainder 2
reverse -> 21
```

Verification:
```text
2*62 + 1 = 125
```

C#:
```csharp
public static string Encode(long number)
{
    if (number == 0) return "0";

    var result = new StringBuilder();

    while (number > 0)
    {
        var remainder = number % 62;
        result.Append(Characters[(int)remainder]);
        number /= 62;
    }

    var chars = result.ToString().ToCharArray();
    Array.Reverse(chars);

    return new string(chars);
}
```

Important:
> Base62 represents an ID; it does not create uniqueness.

---

## 5. Read-heavy workload

A URL may be created once but clicked many times.

Conceptually:
```text
Writes << Reads
```

Therefore the redirect path needs caching.

---

## 6. Redis

Cache:
```text
url:21 -> https://google.com
```

Read path:
```text
GET /21
   |
   v
Redis
 /   \
HIT  MISS
 |     |
302   DB
       |
       v
    Redis SET
       |
       v
      302
```

C#:
```csharp
var key = $"url:{code}";
var originalUrl = await redis.GetStringAsync(key);

if (originalUrl == null)
{
    var id = Base62.Decode(code);
    var mapping = await db.UrlMappings.FindAsync(id);

    if (mapping == null)
        return NotFound();

    originalUrl = mapping.OriginalUrl;
    await redis.SetStringAsync(key, originalUrl);
}

return Redirect(originalUrl);
```

TTL and invalidation depend on whether mappings can change.

---

## 7. API horizontal scaling

```text
                 Users
                   |
                   v
             Load Balancer
              /    |    \
             v     v     v
           API1  API2  API3
              \   |   /
                Redis
```

The API is stateless, so any instance can serve any request.

---

## 8. Unique ID choices

### Identity
```text
API -> DB -> ID
```

### Sequence
```text
API/DB -> Sequence -> ID
```

### GUID
```text
API -> local GUID
```
Distributed but large.

### Snowflake-style
```text
API -> local distributed ID generator -> numeric ID
```

For extreme scale, this avoids using the database as the ID allocator.

---

## 9. Snowflake-style IDs

Common conceptual layout:
```text
1 sign | 41 timestamp | 10 worker | 12 sequence
```

Meaning:
```text
WHEN + WHO + WHICH REQUEST
```

```text
Same ms, worker 1:
sequence 0,1,2,...

Same ms, worker 2:
sequence 0,1,2,...
```

With:
```text
2^10 = 1024 workers
2^12 = 4096 IDs/worker/ms
```

Bit packing:
```text
ID =
(timestamp << 22)
|
(workerId << 12)
|
sequence
```

C# concept:
```csharp
return (timestamp << 22)
     | (_workerId << 12)
     | _sequence;
```

Operational concerns:
- unique worker assignment
- sequence overflow
- clock rollback
- clock synchronization

---

## 10. Snowflake + Base62

```text
Long URL
   |
   v
Snowflake ID
   |
   v
123456789012345
   |
   v
Base62
   |
   v
aZ91k
```

This is the clean conceptual separation:

```text
ID generator -> uniqueness
Base62       -> compact public representation
```

---

## 11. Database scaling

First:
```text
indexes
query tuning
connection pooling
vertical scaling
```

Then:
```text
read replicas
partitioning
sharding
```

Read replicas:
```text
           Primary
          /       \
         v         v
        R1         R2
```

Writes -> primary.
Reads -> replicas.

Async replication can cause read-after-write lag.

Mitigations:
- populate Redis on write
- primary routing for consistency-sensitive reads
- read-your-write strategy

---

## 12. Database sharding

When one database becomes the write/storage bottleneck:

```text
                 Shard Router
                /      |      \
               v       v       v
              DB1     DB2     DB3
```

A conceptual routing method:
```text
stableHash(id) -> shard
```

Why hash?
```text
Sequential IDs + range shards
        |
        v
Newest shard gets most new writes
```

Hashing distributes point writes more evenly.

---

## 13. Shard trade-offs

Benefits:
- write scaling
- storage scaling
- workload distribution

Costs:
- routing complexity
- cross-shard queries
- cross-shard transactions
- rebalancing
- operational complexity

Scatter-gather:
```text
Query
 / | \
DB1 DB2 DB3
 \  |  /
  merge
```

Avoid this for the hot path where possible.

---

## 14. Redis scaling

Replication:
```text
Primary
 /    \
R1    R2
```

Sharding:
```text
Redis Cluster
 /   |   \
S1  S2   S3
```

Combined:
```text
Shard1: P/R
Shard2: P/R
Shard3: P/R
```

---

## 15. Complete write path

```text
POST /api/urls
       |
       v
Load Balancer
       |
       v
Stateless API
       |
       v
Snowflake ID
       |
       v
Base62 code
       |
       v
Shard Router
       |
       +------> DB Shard 1
       +------> DB Shard 2
       +------> DB Shard 3
       |
       v
Redis SET
       |
       v
Return short URL
```

If using DB-generated identity instead:
```text
API -> primary DB -> generated ID -> Base62 -> Redis
```

---

## 16. Complete read path

```text
GET /aZ91k
      |
      v
Load Balancer
      |
      v
API
      |
      v
Redis
    /   \
  HIT   MISS
   |      |
   |      v
   |   Base62 Decode
   |      |
   |      v
   |   Shard Router
   |      |
   |      v
   |   DB Shard
   |      |
   |      v
   |   Redis SET
   |      |
   +------+
      |
      v
 HTTP 302
      |
      v
Original URL
```

---

## 17. Failure handling

### API failure
Load balancer removes unhealthy instance.

### Redis failure
Do not blindly send all traffic to DB.
Use:
- timeout
- circuit breaker
- DB protection
- Redis failover

### DB shard failure
Promote replica.

### Replica lag
Use primary or Redis for read-after-write requirements.

### Cache stampede
Use single-flight/coalescing, TTL jitter, locks, prewarming.

---

## 18. Rate limiting

Protect creation endpoints:
```text
POST /api/urls
```

Algorithms:
- fixed window
- sliding window
- token bucket
- leaky bucket

Redis can maintain distributed counters/token state.

---

## 19. Security

- HTTPS
- validate URL syntax
- protect against malicious/abusive destinations
- rate limit
- authentication where required
- authorization
- abuse reporting/blocklists
- consider open-redirect and phishing/malware risks

---

## 20. Capacity estimation

Assume for an exercise:
```text
10,000 URL creations/sec
1,000,000 redirects/sec
```

URLs/day:
```text
10,000 * 86,400
= 864,000,000
```

Storage must account for:
- URL length
- row overhead
- indexes
- replication
- backups
- metadata

Do not assume the entire dataset belongs in Redis. Cache the working set.

---

## 21. Observability

Track:
```text
Request rate
P50/P95/P99 latency
Error rate
404 rate
Redis hit ratio
Redis latency
DB latency
DB connections
Replication lag
Shard utilization
Hot keys
Hot shards
```

Distributed tracing:
```text
Request
  |
  +-> API
       |
       +-> Redis
       |
       +-> DB
```

---

## 22. Why the architecture scales

```text
Traffic
  |
  v
Load Balancer
  |
  v
Stateless API fleet
  |
  v
Redis cluster
  |
  | misses
  v
Sharded DB + replicas
```

Each bottleneck has a strategy:
```text
API       -> horizontal scaling
Redis     -> replication/sharding
DB reads  -> replicas/cache
DB writes -> partitioning/sharding
Traffic   -> load balancing/rate limiting
Failures  -> failover/timeouts/circuit breakers
```

---

## 23. Recommended interview evolution

Do not start with the final giant architecture.

Start:
```text
API -> DB
```

Then explain the bottleneck:
```text
Redirects are read-heavy
```

Add:
```text
Redis
```

Then:
```text
Load balancer + API fleet
```

Then:
```text
Read replicas
```

Then, only when justified:
```text
Snowflake-style ID generation
DB sharding
Redis sharding
```

This demonstrates reasoning rather than memorization.

---

## 24. 60-minute Bloomberg structure

```text
0-5     Requirements/assumptions
5-10    APIs/data model
10-20   Basic architecture + flows
20-30   Redis/read scaling
30-40   ID generation + DB scaling
40-50   Sharding/consistency/reliability
50-55   Capacity/security
55-60   Failure cases/trade-offs/final architecture
```

---

## 25. Final architecture

```text
                              USERS
                                |
                                v
                         LOAD BALANCER
                                |
                +---------------+---------------+
                |               |               |
                v               v               v
              API1            API2            API3
                |               |               |
                +---------------+---------------+
                                |
                                v
                     +---------------------+
                     |    REDIS CLUSTER    |
                     |  shards + replicas  |
                     +----------+----------+
                                |
                              MISS
                                |
                                v
                       +----------------+
                       |  SHARD ROUTER  |
                       +-------+--------+
                               |
              +----------------+----------------+
              |                |                |
              v                v                v
          DB SHARD 1       DB SHARD 2       DB SHARD 3
          Primary/R        Primary/R        Primary/R
```

### Final mental model

```text
CREATE:
Long URL
 -> unique ID
 -> Base62
 -> shard
 -> durable DB
 -> Redis
 -> short URL

REDIRECT:
Short code
 -> Redis
 -> on miss: decode -> shard -> DB
 -> Redis
 -> HTTP 302
```

---

## 26. Final checklist

```text
[✓] Requirements
[✓] APIs
[✓] Data model
[✓] Base62
[✓] Identity/Sequence
[✓] GUID
[✓] Snowflake-style IDs
[✓] Redis/cache-aside
[✓] TTL/invalidation
[✓] Cache stampede/hot keys
[✓] API horizontal scaling
[✓] Read replicas
[✓] Replication lag
[✓] DB partitioning
[✓] DB sharding
[✓] Hash/consistent hashing
[✓] Cross-shard trade-offs
[✓] Redis replication/sharding
[✓] Failure handling
[✓] Rate limiting
[✓] Security
[✓] Observability
[✓] Capacity estimation
[✓] Complete read/write flows
```

Next deep-dive topics:
```text
CAP theorem
Consistency models
Reliability/failover
Rate limiting
Capacity estimation
Security
Observability
60-minute Bloomberg mock
```

## Deep-Dive Companion Files

The following topics are covered separately in the same interview-preparation format:

1. [Load Balancer Deep Dive](Bloomberg_URL_Shortener_Load_Balancer_Deep_Dive.md)
2. [Capacity Design Deep Dive](Bloomberg_URL_Shortener_Capacity_Design_Deep_Dive.md)
3. [Observability Deep Dive](Bloomberg_URL_Shortener_Observability_Deep_Dive.md)
4. [Security Deep Dive](Bloomberg_URL_Shortener_Security_Deep_Dive.md)
5. [Reliability & Failure Handling Deep Dive](Bloomberg_URL_Shortener_Reliability_Deep_Dive.md)
6. [Consistency & CAP Deep Dive](Bloomberg_URL_Shortener_Consistency_CAP_Deep_Dive.md)

