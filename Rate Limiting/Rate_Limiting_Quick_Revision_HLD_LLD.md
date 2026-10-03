# Rate Limiting — HLD + LLD Quick Revision

> **Purpose:** Fast interview revision for Rate Limiting.
>
> This is intentionally much shorter than the detailed notes. It focuses on the parts that are most likely to be **used in an actual HLD/LLD interview**.

---

# 1. What Is Rate Limiting?

Rate limiting controls how many requests a client can make within a defined policy.

Example:

```text
User = user123
Limit = 100 requests/minute
```

Requests 1–100:

```text
ALLOW
```

Request 101:

```text
429 Too Many Requests
```

Typical response:

```http
HTTP 429 Too Many Requests
```

---

# 2. Why Do We Need It?

Main reasons:

```text
1. Protect APIs
2. Protect databases/downstream services
3. Prevent abuse
4. Ensure fairness
5. Control expensive operations
6. Handle traffic spikes
```

Example:

```text
100,000 requests/sec
        |
        v
Rate Limiter
        |
        +---- 10,000 allowed
        |
        +---- 90,000 rejected
```

The goal is to prevent uncontrolled traffic from reaching expensive components.

---

# 3. Where Does Rate Limiting Sit?

Most common HLD:

```text
Client
   |
   v
Load Balancer
   |
   v
API Gateway
   |
   +---- Rate Limiter
   |
   v
Services
   |
   v
Database / External Services
```

### Why Gateway?

Because it can reject traffic before it reaches application services.

For business-specific limits, the service itself can also enforce limits.

Production systems can use multiple layers:

```text
Gateway limit
      +
Service limit
      +
Expensive-operation limit
```

---

# 4. What Is the Rate-Limit Key?

The key determines **who or what is being limited**.

Common choices:

```text
IP
User ID
API Key
Tenant
Endpoint
User + Endpoint
Tenant + Endpoint
```

Example:

```text
rate_limit:user123:/trade
```

This allows different policies:

```text
/rates          -> 1000/min
/trade          -> 100/min
/generateReport -> 10/min
```

For authenticated systems, user/API-key/tenant based limiting is often more useful than IP alone.

---

# 5. Basic HLD Flow

```text
             Request
                |
                v
       +------------------+
       | Identify Client  |
       +--------+---------+
                |
                v
       +------------------+
       | Find Policy      |
       +--------+---------+
                |
                v
       +------------------+
       | Check Rate State |
       +--------+---------+
                |
          +-----+-----+
          |           |
        ALLOW        REJECT
          |           |
          v           v
       Service       429
```

---

# 6. Algorithms You Must Know

For interviews, know these four:

```text
1. Fixed Window
2. Sliding Window Log
3. Sliding Window Counter
4. Token Bucket
```

The most important practical one to understand deeply:

> **Token Bucket**

---

# 7. Fixed Window

Example:

```text
100 requests/minute
```

Windows:

```text
12:00:00 - 12:00:59
12:01:00 - 12:01:59
12:02:00 - 12:02:59
```

State:

```text
client
window
count
```

Algorithm:

```text
currentWindow = currentTime / windowSize

if new client:
    create counter

if counter.window != currentWindow:
    reset count

if count >= limit:
    reject

count++

allow
```

---

# 8. Fixed Window C#

```csharp
public class FixedWindowRateLimiter
{
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;

    private readonly Dictionary<string, WindowCounter> _counters = new();

    private readonly object _lock = new();

    public FixedWindowRateLimiter(
        int maxRequests,
        TimeSpan windowSize)
    {
        _maxRequests = maxRequests;
        _windowSize = windowSize;
    }

    public bool AllowRequest(string clientId)
    {
        lock (_lock)
        {
            long currentWindow =
                DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
                / (long)_windowSize.TotalMilliseconds;

            if (!_counters.TryGetValue(clientId, out var counter))
            {
                counter = new WindowCounter
                {
                    Window = currentWindow,
                    Count = 0
                };

                _counters[clientId] = counter;
            }

            if (counter.Window != currentWindow)
            {
                counter.Window = currentWindow;
                counter.Count = 0;
            }

            if (counter.Count >= _maxRequests)
                return false;

            counter.Count++;

            return true;
        }
    }
}

public class WindowCounter
{
    public long Window { get; set; }
    public int Count { get; set; }
}
```

---

# 9. Fixed Window Problem

### Boundary Burst

Limit:

```text
100/minute
```

Client sends:

```text
100 requests at 12:00:59
100 requests at 12:01:00
```

Potentially:

```text
200 requests in ~1 second
```

So fixed window is:

```text
Simple
Cheap
O(1)
```

but has:

```text
Boundary burst
```

---

# 10. Distributed Fixed Window

Local memory fails when there are multiple API instances.

```text
             Load Balancer
            /      |      \
           v       v       v
         API1    API2    API3
           |       |       |
           +-------+-------+
                   |
                   v
                 Redis
```

Without Redis:

```text
API1 -> count 40
API2 -> count 30
API3 -> count 50

Total = 120
```

even though the intended limit is:

```text
100
```

Therefore:

> Distributed rate limiting needs shared state.

---

# 11. Redis Fixed Window

Key:

```text
rate_limit:user123:202610021005
```

Value:

```text
73
```

Use Redis:

```text
INCR
```

because Redis increments are atomic.

C#:

```csharp
long count =
    await _redis.StringIncrementAsync(key);

return count <= _maxRequests;
```

---

# 12. `INCR` + `EXPIRE`

Typical approach:

```text
INCR key
EXPIRE key
```

Problem:

```text
INCR succeeds
      |
      v
Application crashes
      |
      X
EXPIRE never executes
```

The key can remain forever.

Production solution:

> Use a Redis Lua script so `INCR + EXPIRE` execute atomically.

```lua
local count = redis.call('INCR', KEYS[1])

if count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end

return count
```

---

# 13. Sliding Window Log

Instead of one counter, store timestamps:

```text
user123:

10:00:01
10:00:03
10:00:05
10:00:08
```

For every request:

```text
1. now = current time
2. remove timestamps older than window
3. if remaining count >= limit -> reject
4. otherwise add current timestamp
5. allow
```

C#:

```csharp
public class SlidingWindowLogRateLimiter
{
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;

    private readonly Dictionary<string, Queue<DateTimeOffset>> _requests
        = new();

    public bool AllowRequest(string clientId)
    {
        var now = DateTimeOffset.UtcNow;
        var windowStart = now - _windowSize;

        if (!_requests.TryGetValue(clientId, out var queue))
        {
            queue = new Queue<DateTimeOffset>();
            _requests[clientId] = queue;
        }

        while (queue.Count > 0 &&
               queue.Peek() <= windowStart)
        {
            queue.Dequeue();
        }

        if (queue.Count >= _maxRequests)
            return false;

        queue.Enqueue(now);

        return true;
    }
}
```

### Main trade-off

```text
High accuracy
+
No fixed-window boundary problem

BUT

High memory
```

because every request timestamp is stored.

---

# 14. Sliding Window Counter

Stores only:

```text
Previous Count
Current Count
```

Formula:

```text
elapsedRatio =
elapsedTime / windowSize

previousWeight =
1 - elapsedRatio

estimatedCount =
previousCount * previousWeight
+
currentCount
```

Example:

```text
Previous = 80
Current = 20
Halfway through current window
```

Then:

```text
previousWeight = 0.5

estimated =
80 * 0.5 + 20
= 60
```

### Trade-off

```text
Low memory
+
Rolling-window approximation

BUT

Not exact
```

---

# 15. Token Bucket — Most Important

Token Bucket has:

```text
Capacity
Refill Rate
Current Tokens
Last Refill Time
```

Example:

```text
Capacity = 10
Refill Rate = 2 tokens/sec
```

Each request costs:

```text
1 token
```

If token exists:

```text
ALLOW
```

Otherwise:

```text
REJECT
```

---

# 16. Token Bucket Mental Model

```text
             2 tokens/sec
                  |
                  v
          +---------------+
          |   TOKEN       |
          |   BUCKET      |
          |               |
          |  o o o o o    |
          |  o o o        |
          +-------+-------+
                  |
             1 token/request
                  |
                  v
               Request
```

Two key parameters:

```text
Capacity
    -> maximum accumulated burst

Refill Rate
    -> sustained rate
```

---

# 17. Token Bucket Formula

This is the key formula:

```text
tokens =
min(
    capacity,
    tokens + elapsedTime * refillRate
)
```

Then:

```text
if tokens >= 1
{
    tokens--
    allow
}
else
{
    reject
}
```

Example:

```text
capacity = 10
refillRate = 2/sec
```

Bucket becomes empty.

Wait 3 seconds:

```text
3 * 2 = 6 tokens
```

Six requests can be allowed.

---

# 18. Token Bucket C#

```csharp
public class TokenBucket
{
    private readonly int _capacity;
    private readonly double _refillRate;

    private double _tokens;
    private DateTimeOffset _lastRefillTime;

    private readonly object _lock = new();

    public TokenBucket(
        int capacity,
        double refillRate)
    {
        _capacity = capacity;
        _refillRate = refillRate;

        _tokens = capacity;
        _lastRefillTime = DateTimeOffset.UtcNow;
    }

    public bool TryConsume()
    {
        lock (_lock)
        {
            Refill();

            if (_tokens < 1)
                return false;

            _tokens--;

            return true;
        }
    }

    private void Refill()
    {
        var now = DateTimeOffset.UtcNow;

        double elapsedSeconds =
            (now - _lastRefillTime).TotalSeconds;

        double tokensToAdd =
            elapsedSeconds * _refillRate;

        _tokens = Math.Min(
            _capacity,
            _tokens + tokensToAdd);

        _lastRefillTime = now;
    }
}
```

---

# 19. Why Lazy Refill?

We do not need a background timer.

Instead:

```text
last request
     |
     | time passes
     v
next request
     |
     v
calculate elapsed time
     |
     v
calculate tokens
```

Formula:

```text
tokensToAdd =
elapsedSeconds * refillRate
```

This is:

> Lazy refill.

Very efficient.

---

# 20. Token Bucket Thread Safety

Without synchronization:

```text
tokens = 1
```

Two threads can both read:

```text
1
```

and both consume.

Therefore this entire operation must be atomic:

```text
REFILL
   +
CHECK
   +
CONSUME
```

Use:

```csharp
lock (_lock)
{
    Refill();

    if (_tokens < 1)
        return false;

    _tokens--;

    return true;
}
```

---

# 21. Per-Client Token Buckets

Conceptually:

```text
user1 -> bucket1
user2 -> bucket2
user3 -> bucket3
```

Manager:

```csharp
private readonly ConcurrentDictionary<string, TokenBucket>
    _buckets = new();
```

Get bucket:

```csharp
var bucket =
    _buckets.GetOrAdd(
        clientId,
        _ => new TokenBucket(
            capacity,
            refillRate));
```

Then:

```csharp
return bucket.TryConsume();
```

Important:

```text
ConcurrentDictionary
    -> protects bucket lookup/storage

lock inside TokenBucket
    -> protects token state
```

---

# 22. Distributed Token Bucket

Local buckets have the same distributed problem.

Production architecture:

```text
                     Load Balancer
                    /      |      \
                   v       v       v
                 API1    API2    API3
                   \       |       /
                    \      |      /
                     +-----+-----+
                           |
                           v
                         Redis
```

Redis stores:

```text
tokens
lastRefill
```

Example:

```text
rate_limit:user123

tokens = 3
lastRefill = T
```

---

# 23. Distributed Token Bucket Algorithm

On every request:

```text
1. Read tokens
2. Read lastRefill
3. Get current time
4. elapsed = now - lastRefill
5. refill tokens
6. cap at capacity
7. check token availability
8. consume token
9. save tokens
10. save lastRefill
11. return allow/reject
```

Critical requirement:

> Steps 1–10 must be atomic.

---

# 24. Why Redis Lua?

Without atomicity:

```text
API1 -> read tokens = 1
API2 -> read tokens = 1

API1 -> consume
API2 -> consume
```

Two requests may be allowed from one token.

With Lua:

```text
API1 ----\
API2 -----+----> Redis
API3 ----/         |
                   v
             Atomic Lua Script
```

Redis executes the state transition atomically.

---

# 25. Distributed Token Bucket C# Skeleton

```csharp
public async Task<bool> AllowRequestAsync(
    string clientId)
{
    string key =
        $"rate_limit:{clientId}";

    var result =
        await _redis.ScriptEvaluateAsync(
            LuaScript,
            new RedisKey[]
            {
                key
            },
            new RedisValue[]
            {
                now,
                _capacity,
                _refillRate
            });

    return (long)result == 1;
}
```

The Lua script performs:

```text
HGET tokens
HGET lastRefill
calculate elapsed
refill
cap
check
consume
HSET tokens
HSET lastRefill
return 1/0
```

---

# 26. Redis Token Bucket State

A Redis hash can look like:

```text
Key:
rate_limit:user123

Fields:

tokens      -> 8
lastRefill  -> 179...
```

Conceptually:

```text
rate_limit:user123
       |
       +---- tokens = 8
       |
       +---- lastRefill = ...
```

---

# 27. Time in Distributed Rate Limiting

Use:

```csharp
DateTimeOffset.UtcNow
```

instead of:

```csharp
DateTime.Now
```

because local time zones can differ.

Useful APIs:

```csharp
DateTimeOffset.UtcNow
```

Current UTC time.

```csharp
DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
```

Numeric Unix timestamp in milliseconds.

```csharp
now - previous
```

Returns a:

```text
TimeSpan
```

```csharp
elapsed.TotalSeconds
```

Returns total elapsed seconds.

---

# 28. UTC vs Clock Skew

Important interview point:

> UTC does not guarantee that every machine has exactly the same physical clock.

Example:

```text
Real time:
10:00:00.000

Server A:
10:00:00.010

Server B:
09:59:59.990
```

Both are UTC.

But their clocks differ.

This is:

> Clock skew.

NTP reduces the difference, but does not make clocks perfectly identical.

---

# 29. Redis `TIME`

If Redis is already the central rate-limit state authority, it can also provide a central time source.

Lua:

```lua
local time = redis.call('TIME')

local seconds =
    tonumber(time[1])

local microseconds =
    tonumber(time[2])

local nowMs =
    seconds * 1000
    + math.floor(microseconds / 1000)
```

Then all API servers use Redis time for the rate calculation.

Mental model:

```text
API1 ----\
API2 -----+----> Redis
API3 ----/         |
                   +--> State
                   |
                   +--> Time
```

Good interview explanation:

> `UtcNow` solves time-zone differences. Redis `TIME` can additionally provide one central clock for the distributed rate-limit calculation.

---

# 30. Rate Limit Response

For rejected requests:

```http
429 Too Many Requests
```

Useful metadata:

```http
Retry-After: 5
```

Possible additional headers:

```text
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
```

Example:

```text
Limit = 100
Remaining = 12
```

This helps clients back off correctly.

---

# 31. Redis Failure

Important HLD decision:

## Fail Open

Redis unavailable:

```text
ALLOW
```

Pros:

```text
higher availability
```

Risk:

```text
backend can receive uncontrolled traffic
```

---

## Fail Closed

Redis unavailable:

```text
REJECT
```

Pros:

```text
stronger protection
```

Risk:

```text
legitimate requests can fail
```

Interview answer:

> The choice depends on the endpoint. A highly available read API may prefer fail-open with additional local protection, while an expensive operation may prefer fail-closed.

---

# 32. TTL / Cleanup

Distributed state cannot remain forever.

Use:

```text
Redis TTL
```

for inactive clients.

For example:

```text
rate_limit:user123
TTL = some inactivity period
```

When inactive:

```text
key expires
```

This prevents unbounded Redis memory growth.

---

# 33. Hot Key

A popular client can create:

```text
rate_limit:user123
```

with extremely high request volume.

This becomes a:

> Hot key.

Possible techniques:

```text
Redis Cluster
local pre-filtering
hierarchical rate limiting
quota allocation
```

But splitting one user's state across independent keys can weaken strict global enforcement.

---

# 34. Multi-Region

Example:

```text
India Region
   |
   +-- API
   +-- Redis

US Region
   |
   +-- API
   +-- Redis
```

If the limit is globally:

```text
100 requests/minute
```

regional independent counters may allow:

```text
India -> 100
US    -> 100
```

Total:

```text
200
```

Possible designs:

```text
1. Central global limiter
2. Regional limits
3. Distributed quota allocation
4. Hierarchical limits
```

Trade-off:

```text
Strict global consistency
        vs
Latency + availability
```

---

# 35. Hierarchical Rate Limiting

High-scale systems can combine local and global limits.

```text
Request
   |
   v
Local Limiter
   |
   v
Global/Redis Limiter
   |
   v
Service
```

Local limiter reduces Redis traffic.

But:

```text
More local caching
       ->
Less Redis traffic
       ->
Harder strict global enforcement
```

This is a trade-off.

---

# 36. Observability

Important metrics:

```text
requests_allowed
requests_rejected
429_count
limiter_latency
redis_latency
redis_errors
hot_keys
```

Useful dimensions:

```text
endpoint
tenant
service
region
algorithm
```

Avoid unnecessarily high-cardinality monitoring labels such as every raw user ID.

---

# 37. Security

Never blindly trust client-provided identity.

Bad:

```http
X-User-Id: user123
```

if the client can change it.

Use trusted identity from:

```text
Authentication token
API key
Gateway authentication
Validated tenant/user context
```

For IP-based limiting, only trust forwarding headers from configured trusted proxies.

---

# 38. HLD Decision Table

| Requirement | Typical Choice |
|---|---|
| Very simple limit | Fixed Window |
| Exact rolling window | Sliding Window Log |
| Rolling approximation + low memory | Sliding Window Counter |
| Controlled burst + sustained rate | Token Bucket |
| Multiple API servers | Redis/shared state |
| Atomic distributed state | Redis Lua |
| Inactive-state cleanup | TTL |
| Multi-region strict global limit | Central/global coordination |
| Very high throughput | Local + global/hierarchical design |

---

# 39. LLD Design

Core abstraction:

```csharp
public interface IRateLimiter
{
    bool AllowRequest(string clientId);
}
```

Implementations:

```text
IRateLimiter
     |
     +-- FixedWindowRateLimiter
     |
     +-- SlidingWindowLogRateLimiter
     |
     +-- SlidingWindowCounterRateLimiter
     |
     +-- TokenBucketRateLimiter
```

This is the:

> Strategy Pattern.

The algorithm can be changed without changing the caller.

---

# 40. Factory

```csharp
public enum RateLimitAlgorithm
{
    FixedWindow,
    SlidingWindowLog,
    SlidingWindowCounter,
    TokenBucket
}
```

Factory concept:

```csharp
public class RateLimiterFactory
{
    public IRateLimiter Create(
        RateLimitAlgorithm algorithm)
    {
        return algorithm switch
        {
            RateLimitAlgorithm.FixedWindow =>
                new FixedWindowRateLimiter(),

            RateLimitAlgorithm.SlidingWindowLog =>
                new SlidingWindowLogRateLimiter(),

            RateLimitAlgorithm.SlidingWindowCounter =>
                new SlidingWindowCounterRateLimiter(),

            RateLimitAlgorithm.TokenBucket =>
                new TokenBucketRateLimiter(),

            _ =>
                throw new ArgumentOutOfRangeException()
        };
    }
}
```

In production, use dependency injection for Redis/configuration rather than manually constructing dependencies in the factory.

---

# 41. Better LLD Result

Instead of returning only:

```csharp
bool
```

use:

```csharp
public class RateLimitResult
{
    public bool Allowed { get; init; }

    public int Limit { get; init; }

    public int? Remaining { get; init; }

    public TimeSpan? RetryAfter { get; init; }
}
```

Then the API layer can produce:

```text
429
Retry-After
Rate-limit headers
```

---

# 42. Actual HLD Flow to Draw in Interview

Use this:

```text
                         CLIENT
                            |
                            v
                    +---------------+
                    | Load Balancer |
                    +-------+-------+
                            |
                            v
                    +---------------+
                    | API Gateway   |
                    +-------+-------+
                            |
                     Rate Limiter
                            |
                     +------+------+
                     |             |
                   ALLOW         REJECT
                     |             |
                     v             v
                 Service          429
                     |
                     v
              Downstream Services
                     |
                     v
                 Database


                 Shared State
                     ^
                     |
                  +------+
                  |Redis |
                  +------+
```

---

# 43. Actual Token Bucket HLD

For a distributed token bucket:

```text
                    CLIENT
                       |
                       v
                 Load Balancer
                       |
            +----------+----------+
            |          |          |
            v          v          v
          API1       API2       API3
            \          |          /
             \         |         /
              +--------+--------+
                       |
                       v
                    Redis
                       |
                 Lua Script
                       |
        +--------------+--------------+
        |              |              |
      tokens       lastRefill       time
                       |
                       v
                Allow / Reject
```

---

# 44. Token Bucket Interview Example

Say:

```text
Capacity = 100
Refill Rate = 10/sec
```

Explain:

```text
100
```

is the maximum burst if the bucket is full.

And:

```text
10/sec
```

is the sustained rate.

If bucket has:

```text
0 tokens
```

and 3 seconds pass:

```text
0 + 3*10
= 30
```

Now 30 requests can be accepted.

This is the cleanest way to explain Token Bucket.

---

# 45. Fixed Window vs Token Bucket

### Fixed Window

```text
100/minute
```

means:

```text
Window 1 -> max 100
Window 2 -> max 100
```

Potential boundary burst:

```text
100 + 100
```

---

### Token Bucket

```text
capacity = 100
refill = 10/sec
```

Allows:

```text
up to 100 burst
```

and then:

```text
10/sec sustained
```

This gives much more natural traffic shaping.

---

# 46. The Most Important Concurrency Rule

Whenever the operation is:

```text
READ
CHECK
UPDATE
```

ask:

> Is this entire operation atomic?

Examples:

### Local Token Bucket

```text
lock
```

### Redis Fixed Window

```text
INCR
```

or:

```text
Lua
```

### Redis Token Bucket

```text
Lua
```

This is one of the most important interview concepts.

---

# 47. The Most Important Distributed-System Rule

Local memory:

```text
API1 -> state1
API2 -> state2
API3 -> state3
```

does NOT give a global rate limit.

Shared state:

```text
API1 ----\
API2 -----+----> Redis
API3 ----/
```

does.

Therefore:

> Horizontal scaling changes where rate-limit state must live.

---

# 48. The Most Important Token Bucket Rule

Remember:

```text
CAPACITY
    =
BURST SIZE

REFILL RATE
    =
SUSTAINED RATE
```

If interviewer asks:

> How would you allow short bursts but control average traffic?

Answer:

> Token Bucket.

---

# 49. The Most Important Redis Rule

If you need:

```text
read
+
calculate
+
check
+
write
```

do not blindly perform separate Redis calls.

Think:

```text
Redis Lua
```

because you need an atomic state transition.

---

# 50. The Most Important Time Rule

Remember:

```text
DateTime.Now
    -> local time

DateTimeOffset.UtcNow
    -> UTC reference

TimeSpan
    -> duration

.TotalSeconds
    -> total duration in seconds

ToUnixTimeMilliseconds()
    -> numeric timestamp

Redis TIME
    -> central Redis time source
```

And:

```text
UTC != perfectly synchronized physical clocks
```

---

# 51. Interview Answer Template

If asked:

> Design a distributed rate limiter.

Start:

### Step 1 — Requirement

```text
What are we limiting?
User / API key / IP / tenant?
```

### Step 2 — Policy

```text
100 req/min?
10 req/sec?
Burst allowed?
```

### Step 3 — Algorithm

```text
Fixed Window?
Sliding Window?
Token Bucket?
```

### Step 4 — Placement

```text
API Gateway
+
possibly service level
```

### Step 5 — Distributed state

```text
Redis
```

### Step 6 — Atomicity

```text
INCR for simple counters
Lua for multi-step state transitions
```

### Step 7 — Failure

```text
Fail-open or fail-closed
based on endpoint requirements
```

### Step 8 — Scaling

```text
TTL
hot keys
Redis Cluster
local/hierarchical limiting
```

### Step 9 — Multi-region

```text
regional vs global limits
consistency vs latency
```

### Step 10 — Observability

```text
allowed
rejected
429
Redis latency
errors
```

---

# 52. 30-Second Revision

If you have only 30 seconds:

```text
Rate Limiting
    |
    +-- Controls request rate
    |
    +-- Usually at Gateway
    |
    +-- Key = user/IP/API-key/tenant/endpoint
    |
    +-- Algorithms:
    |      Fixed Window
    |      Sliding Log
    |      Sliding Counter
    |      Token Bucket
    |
    +-- Distributed:
    |      Redis
    |
    +-- Atomicity:
    |      INCR / Lua
    |
    +-- Token Bucket:
    |      capacity = burst
    |      refillRate = sustained rate
    |
    +-- TTL:
    |      cleanup
    |
    +-- Failure:
    |      fail-open / fail-closed
    |
    +-- Scale:
    |      hot keys / cluster / hierarchy
    |
    +-- Time:
           UTC
           clock skew
           Redis TIME
```

---

# 53. Final Interview Cheat Sheet

## Rate Limiter

```text
Request
   |
Identify client
   |
Find policy
   |
Check/update state
   |
ALLOW / 429
```

## Fixed Window

```text
count + window
```

Problem:

```text
boundary burst
```

## Sliding Log

```text
timestamps
```

Problem:

```text
memory
```

## Sliding Counter

```text
previous + current count
```

Problem:

```text
approximation
```

## Token Bucket

```text
tokens
capacity
refillRate
lastRefill
```

Formula:

```text
tokens =
min(capacity,
    tokens + elapsed * refillRate)
```

Then:

```text
tokens >= 1
    -> consume + allow

tokens < 1
    -> reject
```

## Distributed

```text
API servers
    |
    v
Redis
```

## Atomicity

```text
Simple increment
    -> INCR

Complex read-modify-write
    -> Lua
```

## Time

```text
UTC
+
clock skew awareness
+
Redis TIME if central time is required
```

## HLD

```text
Client
 -> LB
 -> Gateway
 -> Rate Limiter
 -> Service
 -> DB
```

## LLD

```text
IRateLimiter
    |
    +-- Fixed Window
    +-- Sliding Log
    +-- Sliding Counter
    +-- Token Bucket

Strategy Pattern
+
Factory
+
DI
+
Thread Safety
```

---

# One-Line Memory Tricks

```text
Fixed Window
= simplest counter

Sliding Log
= remember every request

Sliding Counter
= remember two counts

Token Bucket
= store permission to send

Capacity
= burst

Refill Rate
= sustained rate

Redis
= shared distributed state

Lua
= atomic multi-step operation

TTL
= cleanup

UTC
= common time reference

Clock Skew
= machines' clocks differ

Redis TIME
= central Redis clock

429
= rate limit rejection
```

---

# End
