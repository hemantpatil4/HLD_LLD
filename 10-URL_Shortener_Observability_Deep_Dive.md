# Bloomberg URL Shortener — Observability Deep Dive

## 1. What is observability?

Observability means being able to understand what is happening inside the system from its external signals.

The three major pillars are:

```text
Logs
Metrics
Traces
```

A useful addition is:

```text
Alerts
```

---

## 2. Why do we need it?

Imagine users report:

> "Short URLs are slow."

Without observability:

```text
??? API?
??? Redis?
??? Database?
??? Network?
```

With observability:

```text
Request latency = 850 ms
        |
        +--> API = 20 ms
        +--> Redis = 5 ms
        +--> DB = 800 ms
```

Now the bottleneck is visible.

---

## 3. Metrics

Metrics are numerical measurements over time.

Important URL-shortener metrics:

### API

```text
Request count
Request rate
Latency
Error rate
CPU
Memory
Thread pool utilization
```

### Redis

```text
Hit ratio
Miss ratio
Latency
Memory usage
Evictions
Connections
Commands/sec
Replication health
```

### Database

```text
Query latency
QPS
CPU
IOPS
Connections
Lock waits
Replication lag
Storage
```

---

## 4. Latency percentiles

Do not rely only on average latency.

Example:

```text
Average = 30 ms
p50     = 20 ms
p95     = 60 ms
p99     = 400 ms
```

Average looks fine, but 1% of requests take much longer.

Common interview metrics:

```text
p50
p95
p99
```

For a user-facing redirect system, tail latency is important.

---

## 5. Logs

Example structured log:

```json
{
  "event": "url_redirect",
  "code": "aZ91k",
  "cacheHit": true,
  "statusCode": 302,
  "durationMs": 4
}
```

Prefer structured logs over arbitrary text.

Useful fields:

- Timestamp
- Request ID / correlation ID
- Endpoint
- Status code
- Duration
- Instance
- Cache hit/miss
- Error information

Do not log sensitive data unnecessarily.

---

## 6. Distributed tracing

Suppose:

```text
Client
 |
Load Balancer
 |
API
 |
Redis
 |
DB
```

A trace follows one request through the system.

```text
Trace ID = abc123

API       20ms
Redis      4ms
DB       200ms
Total    224ms
```

This helps locate latency across services.

---

## 7. Correlation ID

A request can have an identifier:

```text
X-Correlation-ID: abc123
```

Then logs across components can reference:

```text
abc123
```

Example:

```text
API log:
abc123 -> Redis miss

DB log:
abc123 -> SELECT Id=125

API log:
abc123 -> 302
```

This makes debugging much easier.

---

## 8. Health checks

Useful endpoints:

```http
GET /health/live
GET /health/ready
```

Liveness:

```text
Is the process alive?
```

Readiness:

```text
Should traffic be sent here?
```

---

## 9. Alerts

Alert on symptoms that require action.

Examples:

```text
Error rate > threshold
p99 latency > threshold
Redis hit ratio suddenly drops
DB CPU > threshold
DB connection pool exhausted
Replication lag > threshold
Disk almost full
API instance count unexpectedly low
```

Avoid alerting on every small fluctuation.

---

## 10. RED method

For services, a useful model is:

```text
R = Rate
E = Errors
D = Duration
```

Example:

```text
Rate     = 1M RPS
Errors   = 0.1%
Duration = p99 80 ms
```

---

## 11. USE method

For infrastructure:

```text
U = Utilization
S = Saturation
E = Errors
```

Example:

```text
CPU utilization = 80%
DB connections = saturated
Network errors = increasing
```

---

## 12. Cache observability

Suppose Redis hit ratio suddenly changes:

```text
99% -> 70%
```

Database traffic may become:

```text
10K/sec -> 300K/sec
```

This could overload the database.

Therefore cache hit ratio is not just a Redis metric.

It is a system-level capacity signal.

---

## 13. Failure investigation example

Users report slow redirects.

Observe:

```text
API p99 = 900 ms
Redis p99 = 5 ms
DB p99 = 850 ms
```

Then:

```text
API
 |
 Redis  -> healthy
 |
 DB     -> bottleneck
```

Possible DB causes:

- Slow query
- Missing index
- Lock contention
- CPU saturation
- I/O pressure
- Connection exhaustion
- Replica lag

---

## 14. URL Shortener dashboard

A useful dashboard might show:

```text
API
--------------------------------
RPS
p50 / p95 / p99 latency
5xx rate
4xx rate

Redis
--------------------------------
Hit ratio
Commands/sec
Memory
Evictions
Latency

Database
--------------------------------
QPS
CPU
IOPS
Connections
Replication lag
Storage

Infrastructure
--------------------------------
CPU
Memory
Network
Instance count
```

---

## 15. Interview answer

> "I would instrument the system with metrics, structured logs and distributed traces. At the API layer I'd monitor rate, errors and p95/p99 latency. For Redis I'd monitor hit ratio, latency, memory, evictions and replication health. For the database I'd monitor query latency, QPS, connections, CPU, storage and replica lag. Correlation IDs and tracing would let me follow a slow redirect across API, Redis and DB."

---

## 16. Key takeaway

```text
Observability
   |
   +--> Metrics -> What is happening?
   |
   +--> Logs    -> What happened?
   |
   +--> Traces  -> Where did it happen?
   |
   +--> Alerts  -> When do we need action?
```

Without observability, scaling and reliability decisions become guesswork.
