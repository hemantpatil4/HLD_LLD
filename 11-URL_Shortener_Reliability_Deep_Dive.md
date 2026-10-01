# Bloomberg URL Shortener — Reliability & Failure Handling Deep Dive

## 1. What is reliability?

Reliability means the system continues to perform its intended function despite failures.

Failures are normal:

```text
API instance dies
Redis node dies
Database replica fails
Network becomes slow
Deployment fails
Dependency times out
Traffic spikes
```

A good system expects failure instead of assuming everything is always healthy.

---

## 2. Failure domains

Think about different layers:

```text
User
 |
Load Balancer
 |
API
 |
Redis
 |
Database
```

Any layer can fail.

The design should avoid one failure taking down the entire system.

---

## 3. API instance failure

Suppose:

```text
API1 -> DOWN
API2 -> UP
API3 -> UP
```

Load Balancer removes API1.

```text
              Load Balancer
               /          \
              v            v
            API2         API3
```

Because the API is stateless, traffic can move to another instance.

---

## 4. Redis failure

Redis is a cache, not the source of truth.

If Redis is unavailable:

```text
GET
 |
Redis unavailable
 |
DB
 |
302
```

But this is dangerous at high traffic.

If 1M RPS suddenly hits the DB:

```text
Redis failure
     |
     v
1M DB requests/sec
     |
     v
Database overload
```

Therefore graceful degradation is required.

---

## 5. Protecting the database

Useful controls:

### Timeouts

Do not allow requests to wait forever.

### Circuit breaker

If a dependency is repeatedly failing:

```text
API -> DB
       X
```

The circuit can open temporarily instead of sending unlimited traffic.

### Bounded concurrency

Limit how many expensive DB operations are allowed concurrently.

### Rate limiting

Reduce incoming load.

### Backpressure

Slow or reject work when downstream capacity is exhausted.

---

## 6. Retry carefully

Suppose DB request fails.

Naive design:

```text
Request
  |
DB fails
  |
retry
  |
DB fails
  |
retry
  |
retry forever
```

This can make an outage worse.

Better:

```text
Timeout
   |
Limited retry
   |
Exponential backoff
   |
Stop
```

Retries should only be used when the operation is safe to retry.

---

## 7. Retry storm

Suppose 10,000 requests simultaneously fail.

Each retries 3 times:

```text
10,000 original
+
30,000 retries
=
40,000 DB requests
```

This can overload a recovering dependency.

Therefore use:

- Exponential backoff
- Jitter
- Retry limits
- Circuit breakers
- Bulkheads

---

## 8. Database primary failure

Architecture:

```text
              Primary
              /     \
             v       v
        Replica1   Replica2
```

If primary fails, a failover mechanism can promote an appropriate replica.

```text
Old Primary -> FAILED

Replica1 -> New Primary
```

The exact failover mechanism depends on the database platform.

---

## 9. Read replica failure

Suppose:

```text
Replica1 -> DOWN
Replica2 -> UP
```

The read routing layer removes Replica1.

Reads continue through Replica2 if capacity allows.

---

## 10. Replication lag

Async replication:

```text
Primary
  |
  | changes
  v
Replica
```

There can be a delay.

Example:

```text
POST -> Primary
GET  -> Replica
```

Immediately after the POST, the replica may not have the row.

Possible result:

```text
GET -> 404
```

even though the write succeeded.

This is a consistency issue, not necessarily a system failure.

Solutions:

```text
Write-through cache
Read from primary for consistency-sensitive requests
Session/read-your-write routing
Stronger replication guarantees where needed
```

---

## 11. Cache stampede

Suppose a popular key expires:

```text
url:aZ91k
```

Millions of requests see:

```text
MISS
```

and all hit DB.

```text
        1M requests
             |
             v
           Redis
             |
           MISS
             |
       +-----+-----+
       |     |     |
      DB    DB    DB
```

Mitigations:

- TTL jitter
- Request coalescing
- Single-flight
- Refresh-ahead
- Locking for expensive regeneration

---

## 12. Hot key

One URL becomes extremely popular:

```text
url:aZ91k
```

Millions of requests hit the same cache key.

Even if Redis is healthy, one hot key can create concentrated load.

Possible techniques:

- CDN/edge caching
- Local in-memory cache for extremely hot entries
- Replicated/cache distribution strategies
- Request coalescing

---

## 13. Timeout design

Every network dependency should have a bounded timeout.

Example:

```text
API timeout
Redis timeout
DB timeout
```

Do not let:

```text
DB request
   |
   X
   |
wait forever
```

because blocked requests consume resources.

---

## 14. Graceful degradation

If Redis fails:

```text
Normal:
API -> Redis -> 302

Degraded:
API -> DB -> 302
```

But because DB capacity is limited, the system should also:

```text
Rate limit
Bound concurrency
Use circuit breakers
Protect critical resources
```

The goal is not simply "keep everything running."

The goal is:

> Fail in a controlled way without causing cascading failure.

---

## 15. Idempotency

For create operations, retries can be tricky.

Suppose:

```text
Client -> POST
API -> DB INSERT
DB -> success
Network response lost
```

Client retries:

```text
POST again
```

Without an idempotency strategy, two URLs may be created.

Possible solution:

```text
Idempotency-Key: abc123
```

Store the result associated with the key.

Then:

```text
Retry with abc123
       |
       v
Return original result
```

---

## 16. Disaster recovery

Reliability also includes larger failures.

Consider:

```text
Database backup
Replication
Multi-zone deployment
Failover
Restore testing
```

Important metrics/concepts:

### RPO

Recovery Point Objective:

> How much data loss can we tolerate?

### RTO

Recovery Time Objective:

> How quickly must service recover?

---

## 17. Multi-zone architecture

```text
             Load Balancer
             /           \
            v             v
         Zone A         Zone B
        API/Redis      API/Redis
            \             /
             \           /
              Database
```

This reduces dependence on one availability zone.

---

## 18. Reliability interview answer

> "I assume components will fail. The API is stateless so the Load Balancer can route around failed instances. Redis has replicas/failover, but because it is a cache I can degrade to the database with strict protection mechanisms rather than allowing a cache outage to overload the DB. I use timeouts, bounded retries with backoff and jitter, circuit breakers, rate limiting and concurrency limits. For the database I use replication and failover, and I explicitly handle replication lag and read-after-write requirements."

---

## 19. Key takeaway

```text
Reliability
    |
    +--> Redundancy
    +--> Health checks
    +--> Timeouts
    +--> Retries + backoff
    +--> Circuit breakers
    +--> Rate limiting
    +--> Backpressure
    +--> Failover
    +--> Backups
    +--> Disaster recovery
```

The most important principle:

> A failure should not automatically become a cascading failure.
