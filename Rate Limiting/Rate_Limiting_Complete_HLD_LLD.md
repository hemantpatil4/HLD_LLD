# Rate Limiting — Complete HLD + LLD Study Notes

> **Purpose:** Interview-ready, deeply detailed notes for Rate Limiting in High-Level Design (HLD) and Low-Level Design (LLD).
>
> **Implementation language:** C#
>
> **Scope:** This document captures the rate-limiting discussion step by step, including fundamentals, algorithms, examples, flows, concurrency, distributed systems, Redis, Lua atomicity, token bucket, UTC time, clock skew, and design trade-offs.

---

# Table of Contents

1. [What Is Rate Limiting?](#1-what-is-rate-limiting)
2. [Why Do We Need Rate Limiting?](#2-why-do-we-need-rate-limiting)
3. [Real-World Examples](#3-real-world-examples)
4. [Rate Limiting vs Throttling vs Backpressure](#4-rate-limiting-vs-throttling-vs-backpressure)
5. [Where Should Rate Limiting Be Implemented?](#5-where-should-rate-limiting-be-implemented)
6. [What Should We Rate Limit By?](#6-what-should-we-rate-limit-by)
7. [Basic Request Flow](#7-basic-request-flow)
8. [Hard Limit vs Soft Limit](#8-hard-limit-vs-soft-limit)
9. [Fixed Window Counter](#9-fixed-window-counter)
10. [Fixed Window C# Implementation](#10-fixed-window-c-implementation)
11. [Understanding `currentWindow` in Detail](#11-understanding-currentwindow-in-detail)
12. [Fixed Window Concurrency Problem](#12-fixed-window-concurrency-problem)
13. [Thread-Safe Fixed Window](#13-thread-safe-fixed-window)
14. [Why `ConcurrentDictionary` Alone Is Not Enough](#14-why-concurrentdictionary-alone-is-not-enough)
15. [Fixed Window Boundary Burst Problem](#15-fixed-window-boundary-burst-problem)
16. [Why Fixed Window Is Still Useful](#16-why-fixed-window-is-still-useful)
17. [Fixed Window in a Distributed System](#17-fixed-window-in-a-distributed-system)
18. [Redis Fixed Window](#18-redis-fixed-window)
19. [Redis `INCR` and Atomicity](#19-redis-incr-and-atomicity)
20. [`INCR` + `EXPIRE` Problem](#20-incr--expire-problem)
21. [Redis Lua Script Solution](#21-redis-lua-script-solution)
22. [Sliding Window Log](#22-sliding-window-log)
23. [Sliding Window Log C# Implementation](#23-sliding-window-log-c-implementation)
24. [Sliding Window Counter](#24-sliding-window-counter)
25. [Sliding Window Counter Formula](#25-sliding-window-counter-formula)
26. [Sliding Window Counter C# Implementation](#26-sliding-window-counter-c-implementation)
27. [Comparing Fixed Window, Sliding Log, Sliding Counter](#27-comparing-fixed-window-sliding-log-sliding-counter)
28. [Token Bucket](#28-token-bucket)
29. [Token Bucket Mental Model](#29-token-bucket-mental-model)
30. [Token Bucket Mathematics](#30-token-bucket-mathematics)
31. [Token Bucket C# Implementation](#31-token-bucket-c-implementation)
32. [Lazy Refill](#32-lazy-refill)
33. [Date and Time Concepts Needed for Token Bucket](#33-date-and-time-concepts-needed-for-token-bucket)
34. [Token Bucket Thread Safety](#34-token-bucket-thread-safety)
35. [Per-Client Token Buckets](#35-per-client-token-buckets)
36. [Distributed Token Bucket](#36-distributed-token-bucket)
37. [Redis Token Bucket State](#37-redis-token-bucket-state)
38. [Distributed Token Bucket Race Condition](#38-distributed-token-bucket-race-condition)
39. [Redis Lua for Distributed Token Bucket](#39-redis-lua-for-distributed-token-bucket)
40. [Why Redis Time Can Be Useful](#40-why-redis-time-can-be-useful)
41. [`DateTimeOffset.UtcNow` vs Clock Skew](#41-datetimeoffsetutcnow-vs-clock-skew)
42. [Redis `TIME`](#42-redis-time)
43. [Complete Rate-Limiting Architecture](#43-complete-rate-limiting-architecture)
44. [Rate-Limit Response Metadata](#44-rate-limit-response-metadata)
45. [Failure Handling](#45-failure-handling)
46. [Memory and Cleanup](#46-memory-and-cleanup)
47. [Hot Keys and Scaling](#47-hot-keys-and-scaling)
48. [Multi-Region Considerations](#48-multi-region-considerations)
49. [Hierarchical / Hybrid Rate Limiting](#49-hierarchical--hybrid-rate-limiting)
50. [CAP and Consistency Considerations](#50-cap-and-consistency-considerations)
51. [Observability](#51-observability)
52. [Security Considerations](#52-security-considerations)
53. [LLD Direction](#53-lld-direction)
54. [Interview Decision Guide](#54-interview-decision-guide)
55. [Interview-Ready Explanation](#55-interview-ready-explanation)
56. [Common Interview Questions](#56-common-interview-questions)
57. [Final Cheat Sheet](#57-final-cheat-sheet)

---

# 1. What Is Rate Limiting?

Rate limiting controls **how many requests a client is allowed to make within a particular rate policy**.

Example:

```text
User: user123
Policy: 100 requests / minute
```

If the user sends:

```text
Request 1
Request 2
...
Request 100
```

all 100 can be accepted.

The next request:

```text
Request 101
```

is rejected if the limit has been exhausted.

Usually the API returns:

```http
HTTP/1.1 429 Too Many Requests
```

The exact response can also include information such as:

```text
Retry-After
X-RateLimit-Limit
X-RateLimit-Remaining
```

---

# 2. Why Do We Need Rate Limiting?

Rate limiting is not only about stopping malicious users.

It protects the whole system.

## 2.1 Protect the application

Suppose an API normally handles:

```text
1,000 requests/sec
```

Suddenly one client sends:

```text
100,000 requests/sec
```

Without protection:

```text
Client
   |
   v
API
   |
   v
Database
```

The application may consume all CPU, memory, threads, connections, or downstream resources.

With rate limiting:

```text
Client
   |
   v
Rate Limiter
   |
   +---- allowed ----> API
   |
   +---- rejected ---> 429
```

The expensive service is protected.

---

## 2.2 Protect the database

Suppose:

```text
GET /customer/{id}
```

causes:

```text
API
 |
 +--> Redis
 |
 +--> SQL Server
 |
 +--> External service
```

If one client sends millions of requests, the database may become the bottleneck.

Rate limiting prevents uncontrolled request volume from reaching it.

---

## 2.3 Prevent abuse

Rate limiting can reduce:

- brute-force attempts
- API scraping
- accidental request loops
- denial-of-service impact
- excessive polling

It is not a complete DDoS solution, but it is one important layer.

---

## 2.4 Fairness

Suppose there are 10 customers sharing an API.

Without limits:

```text
Customer A -> 90% of capacity
Customers B-J -> remaining 10%
```

With per-customer limits:

```text
Customer A -> maximum 20%
Customer B -> maximum 20%
...
```

The exact policy depends on business requirements.

---

## 2.5 Protect expensive operations

Not every API has the same cost.

For example:

```text
GET /health
```

might be cheap.

But:

```text
POST /generate-report
```

might:

- execute SQL
- generate a PDF
- call multiple services
- perform heavy computation

So the expensive endpoint may need a much stricter rate limit.

---

# 3. Real-World Examples

Typical policies:

```text
100 requests/minute/user
```

or:

```text
10 requests/second/IP
```

or:

```text
1,000 requests/minute/API-key
```

or:

```text
10 requests/minute for /generate-report
```

Different APIs can have different limits.

Example:

```text
GET /rates
    1000 req/min

POST /trade
    100 req/min

POST /generate-report
    10 req/min
```

---

# 4. Rate Limiting vs Throttling vs Backpressure

These terms are related but not identical.

## Rate Limiting

Controls how many requests are allowed.

```text
Limit = 100 requests/minute
```

Request 101 may be rejected.

---

## Throttling

Throttling is a broader concept of restricting or slowing usage.

For example:

```text
Client sends too many requests
        |
        v
System slows processing
```

A throttled request may wait rather than immediately fail.

---

## Backpressure

Backpressure happens when a downstream component cannot keep up.

Example:

```text
Producer
   |
   | 100,000 msg/sec
   v
Queue
   |
   | 5,000 msg/sec
   v
Consumer
```

The consumer is slower.

The system needs a mechanism to communicate:

```text
"Slow down."
```

That is backpressure.

### Simple distinction

```text
Rate Limiting -> How much traffic are you allowed to send?

Throttling    -> How much traffic are we willing/able to process?

Backpressure  -> Downstream is overloaded; slow the producer.
```

---

# 5. Where Should Rate Limiting Be Implemented?

There are multiple layers.

## 5.1 Client side

```text
Client
  |
  v
Client-side limiter
  |
  v
Server
```

Useful for user experience.

But it is **not a security boundary** because the client can be modified or bypassed.

---

## 5.2 API Gateway

Very common.

```text
Internet
   |
   v
API Gateway
   |
   +--> Service A
   |
   +--> Service B
   |
   +--> Service C
```

The gateway can enforce global API policies before requests enter the application.

Advantages:

- centralized
- protects all services
- easy to configure
- prevents unnecessary downstream work

---

## 5.3 Service level

A service can also enforce its own limits.

```text
Gateway
   |
   v
Order Service
   |
   v
Rate Limiter
```

Useful for business-specific limits.

---

## 5.4 Multiple layers

A production system can use multiple layers.

```text
                    +----------------+
Internet ---------->| API Gateway   |
                    | Global Limit   |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    | Service        |
                    | User Limit     |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    | Downstream     |
                    | Protection     |
                    +----------------+
```

For example:

```text
Gateway:
10,000 req/sec per IP

Service:
100 req/min per user

Expensive endpoint:
10 req/min per user
```

---

# 6. What Should We Rate Limit By?

The rate-limit key is extremely important.

## 6.1 IP address

```text
rate_limit:ip:10.10.10.10
```

Useful for anonymous traffic.

Problem:

Many legitimate users can share the same IP due to:

- NAT
- corporate networks
- mobile networks
- proxies

---

## 6.2 User ID

```text
rate_limit:user:user123
```

Good for authenticated APIs.

---

## 6.3 API key

```text
rate_limit:apikey:abc123
```

Common for developer APIs.

---

## 6.4 Endpoint

```text
rate_limit:user123:/generate-report
```

Useful when different APIs have different costs.

---

## 6.5 Tenant

For multi-tenant systems:

```text
rate_limit:tenant:tenant123
```

---

## 6.6 Composite key

Often the most useful.

```text
(userId, endpoint)
```

Example:

```text
rate_limit:user123:/trade
```

This allows:

```text
/trade       -> 100/min
/rates       -> 1000/min
/report      -> 10/min
```

---

# 7. Basic Request Flow

At the conceptual level:

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
       | Read Usage State |
       +--------+---------+
                |
                v
       +------------------+
       | Rate Limit Check |
       +--------+---------+
                |
          +-----+-----+
          |           |
        Allow        Reject
          |           |
          v           v
       API/Service   429
```

The exact usage state depends on the algorithm.

---

# 8. Hard Limit vs Soft Limit

## Hard limit

Example:

```text
100 requests/minute
```

After 100:

```text
101 -> reject
```

---

## Soft limit

The system may allow some temporary burst or grace.

For example:

```text
Normal rate: 100/min
Burst capacity: 20
```

This is common with token bucket.

---

# 9. Fixed Window Counter

This is the simplest algorithm.

Suppose:

```text
Limit = 100 requests/minute
```

Divide time into fixed windows:

```text
12:00:00 - 12:00:59
12:01:00 - 12:01:59
12:02:00 - 12:02:59
```

For every client, keep:

```text
window number
request count
```

Example:

```text
user123
    window = 12:05
    count  = 73
```

---

## 9.1 Algorithm

For every request:

### Step 1

Identify the client.

```text
clientId = user123
```

### Step 2

Calculate the current window.

```text
currentWindow = current time / window size
```

### Step 3

Find the client's counter.

### Step 4

If the window changed:

```text
count = 0
window = currentWindow
```

### Step 5

Check the limit.

```text
if count >= limit
    reject
```

### Step 6

Increment.

```text
count++
```

### Step 7

Allow.

---

# 10. Fixed Window C# Implementation

A simple implementation:

```csharp
public class FixedWindowRateLimiter
{
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;

    private readonly Dictionary<string, WindowCounter> _counters = new();

    public FixedWindowRateLimiter(
        int maxRequests,
        TimeSpan windowSize)
    {
        _maxRequests = maxRequests;
        _windowSize = windowSize;
    }

    public bool AllowRequest(string clientId)
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
        {
            return false;
        }

        counter.Count++;

        return true;
    }
}

public class WindowCounter
{
    public long Window { get; set; }

    public int Count { get; set; }
}
```

---

# 10.1 What is stored?

Conceptually:

```text
Dictionary

user1 -> WindowCounter
           Window = 123456
           Count = 73

user2 -> WindowCounter
           Window = 123456
           Count = 21
```

It is basically a small in-memory table.

---

# 11. Understanding `currentWindow` in Detail

The code:

```csharp
long currentWindow =
    DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
    / (long)_windowSize.TotalMilliseconds;
```

looks complicated initially.

Break it down.

---

## 11.1 `DateTimeOffset.UtcNow`

Means:

> Give me the current UTC date/time.

Example:

```text
2026-10-02 22:10:30 UTC
```

---

## 11.2 `ToUnixTimeMilliseconds()`

Converts the date/time into:

> Number of milliseconds since Unix epoch.

Unix epoch:

```text
1970-01-01 00:00:00 UTC
```

Example conceptually:

```text
2026-10-02 22:10:30 UTC
       |
       v
huge number of milliseconds
```

The actual number is not important for understanding the algorithm.

---

## 11.3 `_windowSize.TotalMilliseconds`

Suppose:

```csharp
_windowSize = TimeSpan.FromMinutes(1);
```

Then:

```csharp
_windowSize.TotalMilliseconds
```

is:

```text
60,000
```

---

## 11.4 Integer division

Suppose simplified time is:

```text
15,000 ms
```

and window size:

```text
10,000 ms
```

Then:

```text
15,000 / 10,000 = 1
```

Because both operands are effectively integers here.

So:

```text
0 - 9,999 ms      -> window 0
10,000 - 19,999   -> window 1
20,000 - 29,999   -> window 2
```

That is exactly what we want.

---

## 11.5 Why not `DateTime.Minute`?

You might think:

```csharp
DateTime.UtcNow.Minute
```

But minute goes:

```text
0
1
2
...
59
0
1
...
```

It repeats every hour.

We want a continuous window identifier.

Unix-time division gives us:

```text
Window 100000
Window 100001
Window 100002
...
```

---

## 11.6 The formula

```text
Window Number
    =
Current Unix Time
    /
Window Duration
```

For fixed window rate limiting:

```text
currentWindow = which fixed time bucket are we currently in?
```

---

# 11.7 Understanding the reset

This code:

```csharp
if (counter.Window != currentWindow)
{
    counter.Window = currentWindow;
    counter.Count = 0;
}
```

means:

```text
Is this request in a new time window?
```

If yes:

```text
old count -> discard
new count -> 0
```

Then the current request is counted.

---

# 12. Fixed Window Concurrency Problem

The first implementation has a race condition.

Suppose:

```text
Limit = 100
Current Count = 99
```

Two threads arrive simultaneously.

### Thread A

```text
reads count = 99
```

### Thread B

```text
reads count = 99
```

Both see:

```text
99 < 100
```

Both increment.

Result:

```text
101 requests allowed
```

But the limit was 100.

This is a classic:

```text
Check -> Modify
```

race condition.

---

# 13. Thread-Safe Fixed Window

The simplest solution is a lock.

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
            {
                return false;
            }

            counter.Count++;

            return true;
        }
    }
}
```

The important point:

```text
check + increment
```

must happen atomically.

---

# 13.1 Why a global lock is not ideal

The above solution locks the entire limiter.

Imagine:

```text
User A request
User B request
User C request
User D request
```

All have to wait for:

```text
same global lock
```

That creates unnecessary contention.

A better design can use:

```text
one state object per client
+
one lock per client
```

The principle is:

```text
User A does not need to block User B.
```

This becomes especially important at high throughput.

---

# 14. Why `ConcurrentDictionary` Alone Is Not Enough

You might replace:

```csharp
Dictionary<string, WindowCounter>
```

with:

```csharp
ConcurrentDictionary<string, WindowCounter>
```

That protects dictionary operations.

But this sequence is still a multi-step operation:

```text
read count
check count
increment count
```

`ConcurrentDictionary` does not automatically make this entire business operation atomic.

The important interview statement:

> A concurrent collection makes individual collection operations thread-safe; it does not automatically make a multi-step business transaction atomic.

---

# 15. Fixed Window Boundary Burst Problem

This is the major weakness of fixed window.

Suppose:

```text
Limit = 100 requests/minute
```

Window 1:

```text
12:00:00 - 12:00:59
```

A client sends:

```text
100 requests at 12:00:59
```

Allowed.

At:

```text
12:01:00
```

a new window starts.

The client can immediately send another:

```text
100 requests
```

So in roughly one second:

```text
100 + 100 = 200 requests
```

were accepted.

The client technically respected:

```text
100 requests per fixed window
```

but generated:

```text
200 requests across the boundary
```

This is called the:

> **Boundary burst problem**

---

# 16. Why Fixed Window Is Still Useful

Despite the boundary problem, fixed window has important advantages.

### Memory

Very small.

Only:

```text
window number
count
```

### Time complexity

Approximately:

```text
O(1)
```

per request.

### Implementation

Very simple.

### Distributed implementation

Very easy with Redis.

### Good use cases

When exact smoothing is not required.

For example:

```text
1000 API requests/minute
```

where a temporary boundary burst is acceptable.

---

# 17. Fixed Window in a Distributed System

Now consider:

```text
                    Load Balancer
                    /     |      \
                   /      |       \
                API1     API2     API3
```

Suppose each server has its own:

```csharp
Dictionary<string, WindowCounter>
```

User sends:

```text
API1 -> 40
API2 -> 30
API3 -> 50
```

Each server thinks:

```text
API1: 40 <= 100
API2: 30 <= 100
API3: 50 <= 100
```

Total:

```text
120
```

The global policy was supposed to be:

```text
100
```

This is the fundamental distributed rate-limiter problem:

> Where is the shared source of truth?

---

# 18. Redis Fixed Window

A common architecture:

```text
                 +----------------+
Client -------->| Load Balancer  |
                 +-------+--------+
                         |
            +------------+------------+
            |            |            |
            v            v            v
          API 1        API 2        API 3
            |            |            |
            +------------+------------+
                         |
                         v
                    +---------+
                    |  Redis  |
                    +---------+
```

Redis becomes the shared rate-limit state.

---

## 18.1 Example Redis key

Suppose:

```text
client = user123
window = 202610021005
```

Key:

```text
rate_limit:user123:202610021005
```

Value:

```text
73
```

Meaning:

```text
user123 has made 73 requests
in this fixed window.
```

---

# 18.2 C# Redis implementation

Using StackExchange.Redis:

```csharp
public class RedisFixedWindowRateLimiter
{
    private readonly IDatabase _redis;
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;

    public RedisFixedWindowRateLimiter(
        IDatabase redis,
        int maxRequests,
        TimeSpan windowSize)
    {
        _redis = redis;
        _maxRequests = maxRequests;
        _windowSize = windowSize;
    }

    public async Task<bool> AllowRequestAsync(string clientId)
    {
        long currentWindow =
            DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
            / (long)_windowSize.TotalMilliseconds;

        string key =
            $"rate_limit:{clientId}:{currentWindow}";

        long count =
            await _redis.StringIncrementAsync(key);

        if (count == 1)
        {
            await _redis.KeyExpireAsync(
                key,
                _windowSize);
        }

        return count <= _maxRequests;
    }
}
```

---

# 19. Redis `INCR` and Atomicity

This line:

```csharp
await _redis.StringIncrementAsync(key);
```

maps to Redis `INCR`.

Redis increments are atomic.

If 10 API servers concurrently increment:

```text
0
```

the result will correctly become:

```text
10
```

rather than suffering from the classic:

```text
read -> modify -> write
```

race.

---

# 20. `INCR` + `EXPIRE` Problem

There is a subtle problem.

We do:

```text
INCR
EXPIRE
```

as two separate commands.

Sequence:

```text
Server
  |
  | INCR
  v
Redis
  |
  | success
  v
Server
  |
  X crash
```

The `EXPIRE` command never happens.

Now the key may remain forever.

That creates a memory leak in Redis.

---

# 21. Redis Lua Script Solution

We can make the operations atomic with Lua.

```lua
local count = redis.call('INCR', KEYS[1])

if count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end

return count
```

---

## 21.1 Understanding `KEYS[1]`

When Redis executes:

```text
KEYS = [rate_limit:user123:123]
```

then:

```lua
KEYS[1]
```

means:

```text
rate_limit:user123:123
```

---

## 21.2 Understanding `ARGV[1]`

If the C# application passes:

```text
60
```

then:

```lua
ARGV[1]
```

is:

```text
60
```

seconds.

---

## 21.3 Why Lua solves the problem

Redis executes the script atomically.

Conceptually:

```text
INCR
  +
EXPIRE
```

becomes one atomic operation.

No other Redis command interleaves in the middle of the script.

---

## 21.4 C# script

```csharp
private const string LuaScript = """
    local count = redis.call('INCR', KEYS[1])

    if count == 1 then
        redis.call('EXPIRE', KEYS[1], ARGV[1])
    end

    return count
    """;
```

The script can be executed through:

```csharp
var result =
    await _redis.ScriptEvaluateAsync(
        LuaScript,
        new RedisKey[] { key },
        new RedisValue[]
        {
            ttlSeconds
        });
```

---

# 22. Sliding Window Log

Fixed window has the boundary burst problem.

A more accurate algorithm is:

> Sliding Window Log.

Instead of storing one count, store the timestamp of every request.

For example:

```text
user123

10:00:01
10:00:03
10:00:05
10:00:07
```

If the limit is:

```text
3 requests / 10 seconds
```

we inspect the timestamps in the current 10-second period.

---

# 23. Sliding Window Log C# Implementation

A simple implementation:

```csharp
public class SlidingWindowLogRateLimiter
{
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;

    private readonly Dictionary<string, Queue<DateTimeOffset>> _requests
        = new();

    public SlidingWindowLogRateLimiter(
        int maxRequests,
        TimeSpan windowSize)
    {
        _maxRequests = maxRequests;
        _windowSize = windowSize;
    }

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
        {
            return false;
        }

        queue.Enqueue(now);

        return true;
    }
}
```

---

# 23.1 Why a Queue?

Requests arrive in chronological order.

Example:

```text
10:00:01
10:00:03
10:00:05
10:00:08
```

The oldest request is always at:

```text
queue.Peek()
```

So when it becomes too old:

```csharp
queue.Dequeue();
```

We remove old requests efficiently.

---

# 23.2 Example

Policy:

```text
3 requests / 10 seconds
```

Current time:

```text
10:00:12
```

Window:

```text
10:00:02 -> 10:00:12
```

Suppose queue contains:

```text
10:00:01
10:00:05
10:00:08
```

`10:00:01` is outside the window.

Remove it.

Remaining:

```text
10:00:05
10:00:08
```

Count:

```text
2
```

The next request is allowed.

---

# 23.3 Main advantage

The window continuously moves.

There is no artificial:

```text
12:00:59
12:01:00
```

boundary.

Therefore it is much more accurate than fixed window.

---

# 23.4 Main disadvantage

Memory.

If there are:

```text
1,000,000 users
```

and each user sends many requests, storing every timestamp becomes expensive.

Memory is approximately proportional to:

```text
number of requests in active windows
```

rather than simply:

```text
number of users
```

---

# 24. Sliding Window Counter

Sliding Window Counter is a compromise.

Instead of storing every timestamp, store only:

```text
previous window count
current window count
```

Example:

```text
Limit = 100 requests/minute
```

Store:

```text
Previous Window = 80
Current Window  = 20
```

We estimate how many previous-window requests are still relevant.

---

# 25. Sliding Window Counter Formula

Suppose:

```text
Window size = 60 seconds
```

We are halfway through the current window.

Then:

```text
elapsedRatio = 0.5
previousWeight = 1 - 0.5
               = 0.5
```

Estimated count:

```text
previousCount * previousWeight
+
currentCount
```

So:

```text
80 * 0.5 + 20
= 40 + 20
= 60
```

Estimated current sliding-window count:

```text
60
```

---

## 25.1 General formula

```text
elapsedRatio
    =
elapsed time in current window
/
window size
```

Then:

```text
previousWeight
    =
1 - elapsedRatio
```

And:

```text
estimatedCount
    =
previousCount * previousWeight
+
currentCount
```

---

# 26. Sliding Window Counter C# Implementation

```csharp
public class WindowCounter
{
    public long WindowNumber { get; set; }

    public int CurrentCount { get; set; }

    public int PreviousCount { get; set; }
}
```

Rate limiter:

```csharp
public class SlidingWindowCounterRateLimiter
{
    private readonly int _maxRequests;
    private readonly TimeSpan _windowSize;

    private readonly Dictionary<string, WindowCounter> _counters
        = new();

    public SlidingWindowCounterRateLimiter(
        int maxRequests,
        TimeSpan windowSize)
    {
        _maxRequests = maxRequests;
        _windowSize = windowSize;
    }

    public bool AllowRequest(string clientId)
    {
        DateTimeOffset now = DateTimeOffset.UtcNow;

        long nowMilliseconds =
            now.ToUnixTimeMilliseconds();

        long windowSizeMilliseconds =
            (long)_windowSize.TotalMilliseconds;

        long windowNumber =
            nowMilliseconds / windowSizeMilliseconds;

        long windowStartMilliseconds =
            windowNumber * windowSizeMilliseconds;

        long elapsedMilliseconds =
            nowMilliseconds - windowStartMilliseconds;

        double elapsedRatio =
            (double)elapsedMilliseconds
            / windowSizeMilliseconds;

        if (!_counters.TryGetValue(
                clientId,
                out var counter))
        {
            counter = new WindowCounter
            {
                WindowNumber = windowNumber,
                CurrentCount = 0,
                PreviousCount = 0
            };

            _counters[clientId] = counter;
        }

        if (counter.WindowNumber != windowNumber)
        {
            counter.PreviousCount =
                counter.CurrentCount;

            counter.CurrentCount = 0;

            counter.WindowNumber =
                windowNumber;
        }

        double previousWeight =
            1 - elapsedRatio;

        double estimatedCount =
            counter.PreviousCount * previousWeight
            + counter.CurrentCount;

        if (estimatedCount >= _maxRequests)
        {
            return false;
        }

        counter.CurrentCount++;

        return true;
    }
}
```

---

# 26.1 Important limitation

Sliding Window Counter is an **approximation**.

Why?

Because we don't know the exact distribution of requests inside the previous window.

We only know:

```text
previous window total = 80
```

We don't know whether those 80 requests happened:

```text
all at the beginning
```

or:

```text
all near the end
```

The algorithm assumes a proportional distribution.

That is why it is more memory-efficient but less exact than Sliding Window Log.

---

# 27. Comparing Fixed Window, Sliding Log, Sliding Counter

| Algorithm | State | Accuracy | Memory | Burst Problem | Complexity |
|---|---|---:|---:|---|---|
| Fixed Window | Count | Lower | Very low | Yes | O(1) |
| Sliding Log | Every timestamp | High | High | Much lower | O(1) amortized |
| Sliding Counter | 2 counts | Approximate | Low | Reduced | O(1) |
| Token Bucket | Tokens + timestamp | Policy-oriented | Low | Controlled burst | O(1) |

---

# 27.1 Fixed Window

Use when:

```text
simplicity > precision
```

Example:

```text
1000 requests/minute
```

---

# 27.2 Sliding Window Log

Use when:

```text
accurate rolling-window behavior is important
```

but memory is acceptable.

---

# 27.3 Sliding Window Counter

Use when:

```text
you want rolling-window behavior
without storing every timestamp
```

---

# 27.4 Token Bucket

Especially useful when you want:

```text
controlled bursts
+
defined long-term rate
```

It is one of the most important algorithms to know for system-design interviews.

---

# 28. Token Bucket

Imagine a bucket containing tokens.

Example:

```text
Capacity = 10 tokens
Refill rate = 2 tokens/second
```

Each request costs:

```text
1 token
```

If a token exists:

```text
request -> consume token -> allow
```

If no token exists:

```text
request -> reject
```

---

# 29. Token Bucket Mental Model

Imagine:

```text
          Refill
       2 tokens/sec
             |
             v
       +-------------+
       |  o o o o o  |
       |  o o o      |  capacity = 10
       +-------------+
             |
             | 1 token/request
             v
          Request
```

There are two important parameters.

### Capacity

Controls maximum burst.

```text
capacity = 10
```

means up to 10 tokens can be accumulated.

Therefore a client can potentially send:

```text
10 requests immediately
```

if the bucket is full.

### Refill rate

Controls long-term rate.

```text
2 tokens/sec
```

means approximately:

```text
2 requests/sec
```

can be sustained over time.

---

# 30. Token Bucket Mathematics

The core equation is:

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
    tokens = tokens - 1
    allow
else
    reject
```

---

## 30.1 Example

Configuration:

```text
capacity = 10
refillRate = 2 tokens/sec
```

Initial:

```text
tokens = 10
```

10 requests arrive immediately.

After 10 requests:

```text
tokens = 0
```

Wait 3 seconds.

Refill:

```text
3 * 2 = 6
```

Now:

```text
tokens = 6
```

Six requests can be allowed.

---

## 30.2 Burst vs sustained rate

Token bucket separates two concepts:

```text
Capacity
    -> burst size

Refill rate
    -> sustained rate
```

This is a very useful interview explanation.

---

# 31. Token Bucket C# Implementation

```csharp
public class TokenBucketRateLimiter
{
    private readonly int _capacity;
    private readonly double _refillRate;

    private double _tokens;

    private DateTimeOffset _lastRefillTime;

    public TokenBucketRateLimiter(
        int capacity,
        double refillRate)
    {
        _capacity = capacity;
        _refillRate = refillRate;

        _tokens = capacity;

        _lastRefillTime =
            DateTimeOffset.UtcNow;
    }

    public bool AllowRequest()
    {
        RefillTokens();

        if (_tokens < 1)
        {
            return false;
        }

        _tokens--;

        return true;
    }

    private void RefillTokens()
    {
        DateTimeOffset now =
            DateTimeOffset.UtcNow;

        double elapsedSeconds =
            (now - _lastRefillTime)
            .TotalSeconds;

        double tokensToAdd =
            elapsedSeconds * _refillRate;

        _tokens =
            Math.Min(
                _capacity,
                _tokens + tokensToAdd);

        _lastRefillTime = now;
    }
}
```

---

# 31.1 Why is `_tokens` a `double`?

Suppose:

```text
refillRate = 2 tokens/sec
```

After:

```text
0.25 sec
```

we get:

```text
0.5 tokens
```

We cannot yet consume one complete token.

After another:

```text
0.25 sec
```

we have:

```text
1 token
```

Using `double` lets fractional tokens accumulate naturally.

---

# 32. Lazy Refill

Notice that there is no timer.

We do not continuously execute:

```text
every 1 millisecond
    add tokens
```

Instead, we calculate the refill when a request arrives.

Example:

```text
Last request:
10:00:00

Next request:
10:00:05
```

At the second request:

```text
elapsed = 5 sec
```

Then:

```text
tokensToAdd =
5 * refillRate
```

This is called:

> **Lazy refill**

It is efficient because there is no background process for every bucket.

---

# 33. Date and Time Concepts Needed for Token Bucket

Understanding C# date/time APIs is important here.

---

## 33.1 `DateTime.Now`

Returns local server time.

Example:

```csharp
DateTime.Now
```

If the server is configured for India:

```text
IST
```

Another server may be configured for UTC.

Therefore local times can differ.

---

## 33.2 `DateTime.UtcNow`

Returns UTC time.

```csharp
DateTime.UtcNow
```

Useful when systems communicate across time zones.

---

## 33.3 `DateTimeOffset.Now`

Represents local date/time together with its offset.

Example conceptually:

```text
2026-10-03 03:00:00 +05:30
```

---

## 33.4 `DateTimeOffset.UtcNow`

Represents current UTC time with an explicit offset.

Example:

```text
2026-10-02 21:30:00 +00:00
```

For distributed systems, this is generally preferable to local time when wall-clock timestamps are needed.

---

## 33.5 `TimeSpan`

`TimeSpan` represents a duration.

Example:

```csharp
TimeSpan.FromSeconds(10);
TimeSpan.FromMinutes(1);
TimeSpan.FromHours(1);
```

---

## 33.6 `TotalSeconds`

Example:

```csharp
TimeSpan duration =
    TimeSpan.FromMinutes(2.5);
```

Then:

```csharp
duration.TotalSeconds
```

returns:

```text
150
```

---

## 33.7 `Seconds` vs `TotalSeconds`

This distinction is important.

If:

```text
2 minutes 30 seconds
```

then:

```csharp
duration.TotalSeconds
```

is:

```text
150
```

while:

```csharp
duration.Seconds
```

is:

```text
30
```

`Seconds` is only the seconds component.

`TotalSeconds` is the entire duration converted to seconds.

For token refill we want:

```csharp
.TotalSeconds
```

---

## 33.8 Subtracting times

```csharp
var elapsed =
    now - _lastRefillTime;
```

returns:

```text
TimeSpan
```

Then:

```csharp
elapsed.TotalSeconds
```

tells us how many seconds passed.

---

## 33.9 Unix time

```csharp
DateTimeOffset.UtcNow
    .ToUnixTimeSeconds();
```

or:

```csharp
DateTimeOffset.UtcNow
    .ToUnixTimeMilliseconds();
```

Useful when we want a numeric timestamp.

---

## 33.10 `Stopwatch`

For measuring duration of local code execution:

```csharp
Stopwatch stopwatch = Stopwatch.StartNew();

DoWork();

stopwatch.Stop();

Console.WriteLine(
    stopwatch.ElapsedMilliseconds);
```

`Stopwatch` is designed for elapsed-duration measurement.

For distributed rate-limit state based on wall-clock timestamps, `DateTimeOffset.UtcNow` or a central time source is often used.

---

# 34. Token Bucket Thread Safety

The simple token bucket has a race condition.

Suppose:

```text
tokens = 1
```

Two threads arrive.

Thread A:

```text
reads 1
```

Thread B:

```text
reads 1
```

Both may decide:

```text
1 >= 1
```

Both decrement.

Potentially:

```text
2 requests allowed
```

from one token.

---

## 34.1 Lock solution

```csharp
private readonly object _lock = new();

public bool AllowRequest()
{
    lock (_lock)
    {
        RefillTokens();

        if (_tokens < 1)
        {
            return false;
        }

        _tokens--;

        return true;
    }
}
```

The important point is that:

```text
refill
+
check
+
consume
```

must be one atomic state transition.

---

# 34.2 Full thread-safe bucket

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
        _lastRefillTime =
            DateTimeOffset.UtcNow;
    }

    public bool TryConsume()
    {
        lock (_lock)
        {
            Refill();

            if (_tokens < 1)
            {
                return false;
            }

            _tokens--;

            return true;
        }
    }

    private void Refill()
    {
        var now =
            DateTimeOffset.UtcNow;

        double elapsedSeconds =
            (now - _lastRefillTime)
            .TotalSeconds;

        double tokensToAdd =
            elapsedSeconds * _refillRate;

        _tokens =
            Math.Min(
                _capacity,
                _tokens + tokensToAdd);

        _lastRefillTime = now;
    }
}
```

---

# 35. Per-Client Token Buckets

In a real API we usually need one bucket per client.

```text
user1 -> bucket1
user2 -> bucket2
user3 -> bucket3
```

A manager can use:

```csharp
ConcurrentDictionary<string, TokenBucket>
```

Example:

```csharp
public class TokenBucketRateLimiter
{
    private readonly int _capacity;
    private readonly double _refillRate;

    private readonly ConcurrentDictionary<string, TokenBucket>
        _buckets = new();

    public TokenBucketRateLimiter(
        int capacity,
        double refillRate)
    {
        _capacity = capacity;
        _refillRate = refillRate;
    }

    public bool AllowRequest(string clientId)
    {
        var bucket =
            _buckets.GetOrAdd(
                clientId,
                _ => new TokenBucket(
                    _capacity,
                    _refillRate));

        return bucket.TryConsume();
    }
}
```

---

## 35.1 Why both `ConcurrentDictionary` and `lock`?

They solve different problems.

`ConcurrentDictionary` protects:

```text
clientId -> bucket
```

The lock inside `TokenBucket` protects:

```text
tokens
lastRefillTime
```

The dictionary being thread-safe does not make the bucket state thread-safe.

---

# 36. Distributed Token Bucket

Local memory fails in a multi-server environment.

Architecture:

```text
                         Load Balancer
                       /      |       \
                      /       |        \
                     v        v         v
                   API1     API2      API3
                     \        |        /
                      \       |       /
                       +------v------+
                              |
                              v
                         +---------+
                         |  Redis  |
                         +---------+
```

Redis becomes the shared state.

---

# 37. Redis Token Bucket State

For each client:

```text
rate_limit:user123
```

Store:

```text
tokens
lastRefill
```

Example:

```text
tokens      = 3
lastRefill  = 10:00:00
```

Suppose:

```text
capacity = 10
refillRate = 2/sec
```

A request arrives at:

```text
10:00:03
```

Elapsed:

```text
3 seconds
```

Refill:

```text
3 * 2 = 6
```

New tokens:

```text
3 + 6 = 9
```

Consume one:

```text
8
```

Store:

```text
tokens = 8
lastRefill = 10:00:03
```

---

# 38. Distributed Token Bucket Race Condition

Suppose Redis contains:

```text
tokens = 1
```

Two API servers receive requests.

### API 1

Reads:

```text
tokens = 1
```

### API 2

Reads:

```text
tokens = 1
```

Both calculate:

```text
1 >= 1
```

Both consume.

Both write:

```text
tokens = 0
```

Two requests were allowed even though only one token existed.

This is a classic distributed:

```text
read -> calculate -> write
```

race.

We need:

> Atomic read-modify-write.

---

# 39. Redis Lua for Distributed Token Bucket

The complete state transition should happen atomically inside Redis.

Conceptually:

```text
Read tokens
Read lastRefill
Get current time
Calculate elapsed
Refill
Cap at capacity
Check token availability
Consume
Save state
Return decision
```

No other Redis command should interleave between these operations.

---

## 39.1 C# implementation

```csharp
public class RedisTokenBucketRateLimiter
{
    private readonly IDatabase _redis;

    private readonly int _capacity;
    private readonly double _refillRate;

    public RedisTokenBucketRateLimiter(
        IDatabase redis,
        int capacity,
        double refillRate)
    {
        _redis = redis;
        _capacity = capacity;
        _refillRate = refillRate;
    }

    public async Task<bool> AllowRequestAsync(
        string clientId)
    {
        string key =
            $"rate_limit:{clientId}";

        const string script = """
            local tokens =
                tonumber(
                    redis.call(
                        'HGET',
                        KEYS[1],
                        'tokens'
                    )
                )

            local lastRefill =
                tonumber(
                    redis.call(
                        'HGET',
                        KEYS[1],
                        'lastRefill'
                    )
                )

            local now =
                tonumber(ARGV[1])

            local capacity =
                tonumber(ARGV[2])

            local refillRate =
                tonumber(ARGV[3])

            if tokens == nil then
                tokens = capacity
            end

            if lastRefill == nil then
                lastRefill = now
            end

            local elapsed =
                now - lastRefill

            local newTokens =
                tokens + (elapsed * refillRate)

            if newTokens > capacity then
                newTokens = capacity
            end

            if newTokens < 1 then

                redis.call(
                    'HSET',
                    KEYS[1],
                    'tokens',
                    newTokens,
                    'lastRefill',
                    now
                )

                return 0
            end

            newTokens =
                newTokens - 1

            redis.call(
                'HSET',
                KEYS[1],
                'tokens',
                newTokens,
                'lastRefill',
                now
            )

            return 1
            """;

        long now =
            DateTimeOffset.UtcNow
                .ToUnixTimeMilliseconds();

        var result =
            await _redis.ScriptEvaluateAsync(
                script,
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
}
```

---

# 39.2 Understanding the Redis hash

The key:

```text
rate_limit:user123
```

may contain:

```text
tokens      -> 8
lastRefill  -> 179...
```

Conceptually:

```text
Redis Hash

rate_limit:user123
    |
    +-- tokens = 8
    |
    +-- lastRefill = 179...
```

---

# 39.3 Why Lua?

Because the entire state transition:

```text
read
calculate
check
update
```

must be atomic.

The script executes as one Redis operation.

---

# 39.4 Important detail: fractional tokens

If:

```text
refillRate = 2 tokens/sec
```

and elapsed time is:

```text
0.25 sec
```

then:

```text
0.5 tokens
```

are added.

Therefore Redis state may contain fractional values.

A production implementation can also use fixed-point integer arithmetic if deterministic numeric behavior is required.

---

# 39.5 Redis Cluster consideration

Redis Lua scripts execute atomically on one Redis node.

In Redis Cluster, all keys used by one script must belong to the same hash slot.

For a single bucket key:

```text
rate_limit:user123
```

this is naturally straightforward.

If a future design needs multiple keys in one atomic script, Redis Cluster hash tags may be needed so those keys map to the same slot.

---

# 40. Why Redis Time Can Be Useful

The first distributed implementation passes:

```csharp
DateTimeOffset.UtcNow
```

from the API server.

This works well in many systems.

But there is another option:

```text
Redis itself provides the time.
```

Why?

Because Redis is already the central state authority.

---

# 41. `DateTimeOffset.UtcNow` vs Clock Skew

An important question is:

> If all servers use UTC, why would their times be different?

The answer:

**UTC defines the same time reference, but every server has its own physical system clock.**

Example:

Actual time:

```text
10:00:00.000
```

Server A:

```text
10:00:00.010
```

Server B:

```text
09:59:59.990
```

Both are reporting UTC.

But their clocks differ by:

```text
20 ms
```

This is called:

> **Clock skew**

---

## 41.1 Does NTP solve it?

Network Time Protocol (NTP) continuously synchronizes system clocks.

So in a healthy infrastructure:

```text
Server A -> almost correct
Server B -> almost correct
Server C -> almost correct
```

But they are not mathematically guaranteed to be identical at every instant.

---

## 41.2 What `UtcNow` solves

It solves the **time-zone problem**.

For example:

```text
Server A -> India
Server B -> London
Server C -> US
```

Using local time would make comparison difficult.

Using:

```csharp
DateTimeOffset.UtcNow
```

gives a common time reference.

---

## 41.3 What `UtcNow` does not solve

It does not eliminate:

```text
clock skew
```

between physical machines.

---

# 42. Redis `TIME`

Redis provides its own server time.

Inside Lua:

```lua
local time = redis.call('TIME')
```

Redis returns:

```text
seconds
microseconds
```

Conceptually:

```text
seconds     = Unix seconds
microseconds = additional microseconds
```

You can derive milliseconds:

```lua
local seconds =
    tonumber(time[1])

local microseconds =
    tonumber(time[2])

local nowMs =
    seconds * 1000
    + math.floor(microseconds / 1000)
```

Now the rate-limit calculation uses Redis's time rather than whichever API server received the request.

---

# 42.1 Why this can be useful

Suppose:

```text
API1 clock = slightly ahead
API2 clock = slightly behind
```

Both call:

```text
Redis TIME
```

and Redis provides one central time source.

Architecture becomes:

```text
API1 ----\
           \
API2 ------> Redis
           /
API3 ----/
           |
           +--> authoritative rate-limit time
```

---

# 42.2 Key interview statement

A good explanation is:

> `DateTimeOffset.UtcNow` gives all servers the same UTC reference and is usually sufficient when small clock differences are acceptable. If the distributed rate-limit calculation needs one central time source, Redis `TIME` can be used because Redis is already the shared state authority.

---

# 43. Complete Rate-Limiting Architecture

A production-oriented conceptual architecture:

```text
                       Internet
                           |
                           v
                  +----------------+
                  | Load Balancer  |
                  +-------+--------+
                          |
                          v
                  +----------------+
                  | API Gateway    |
                  |                |
                  | Rate Limiter   |
                  +-------+--------+
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
           Service A   Service B   Service C
              |           |           |
              +-----------+-----------+
                          |
                          v
                     +---------+
                     | Redis   |
                     +---------+
                          |
                          v
                     Rate State
```

---

# 43.1 Request flow

Example:

```text
POST /trade
User = user123
```

### Step 1 — Identify client

```text
user123
```

### Step 2 — Determine policy

```text
/trade
100 requests/minute
```

### Step 3 — Determine bucket/key

For token bucket:

```text
rate_limit:user123:/trade
```

### Step 4 — Read/update rate-limit state

Redis.

### Step 5 — Decision

```text
Allowed
```

or:

```text
Rejected
```

### Step 6 — Continue or return 429

```text
Allowed:
    Gateway -> Service

Rejected:
    Gateway -> 429
```

---

# 44. Rate-Limit Response Metadata

A useful response may include:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 5
```

Meaning:

```text
Try again after approximately 5 seconds.
```

Other useful headers include:

```text
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
```

For example:

```text
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 12
```

These improve client behavior.

---

# 45. Failure Handling

What happens if Redis is unavailable?

This is an important HLD decision.

Two broad options exist.

## 45.1 Fail open

If rate limiter cannot be reached:

```text
allow request
```

Advantages:

- preserves availability
- users are less likely to see false rejections

Risk:

```text
backend may become overloaded
```

---

## 45.2 Fail closed

If rate limiter cannot be reached:

```text
reject request
```

Advantages:

- stronger protection
- prevents uncontrolled traffic from reaching downstream services

Risk:

```text
legitimate traffic may be rejected
```

---

## 45.3 The correct interview answer

Do not say one is universally correct.

Say:

> The choice depends on the endpoint and business requirements. For a highly availability-sensitive read API, fail-open may be acceptable with additional local protection. For an expensive or safety-critical operation, fail-closed may be preferred to protect downstream systems.

---

# 46. Memory and Cleanup

Per-client rate-limit state can grow indefinitely if inactive clients are never removed.

For local memory:

```text
user1 -> bucket
user2 -> bucket
...
millions of users
```

Eventually memory becomes a problem.

Solutions include:

- expiration
- idle eviction
- bounded caches
- periodic cleanup
- distributed storage with TTL

---

# 46.1 Redis TTL

Redis is especially useful because keys can expire automatically.

For example:

```text
rate_limit:user123
```

can have:

```text
TTL = some inactivity period
```

When the client becomes inactive:

```text
key expires
```

and memory is reclaimed.

For token bucket, the TTL should be selected carefully. A useful approach is to keep state alive long enough to cover the time needed for the bucket to refill, plus an operational buffer.

---

# 47. Hot Keys and Scaling

Suppose one client is extremely popular:

```text
user123
```

and every request hits:

```text
rate_limit:user123
```

That state becomes a:

> Hot key.

A single Redis key can receive enormous traffic.

Possible approaches include:

- Redis clustering
- sharding keys
- local pre-filtering
- hierarchical rate limiting
- carefully designed per-node budgets

But blindly splitting one logical user's counter across many independent keys can make a strict global limit difficult to enforce.

---

# 48. Multi-Region Considerations

Suppose:

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

Now one user may send traffic to both regions.

Question:

> Where is the global rate-limit state?

Options include:

### Regional limits

Each region enforces its own limit.

Simple and low latency.

But global usage can exceed the intended limit.

### Centralized global limiter

All regions use one shared authority.

More accurate.

But introduces:

- cross-region latency
- dependency on network connectivity
- global service bottleneck

### Distributed quota allocation

A global quota can be divided:

```text
1000 req/min global

India -> 600
US    -> 400
```

Regions enforce local budgets and periodically rebalance.

This reduces cross-region latency but introduces quota-management complexity.

---

# 49. Hierarchical / Hybrid Rate Limiting

At very high scale, one Redis operation for every request may be expensive.

A hierarchical approach can use:

```text
Client
   |
   v
Local limiter
   |
   v
Distributed/global limiter
   |
   v
Service
```

For example:

```text
Local node budget = 100 requests
Global user budget = 1000 requests
```

The local limiter absorbs some traffic without calling Redis for every request.

However, this introduces an important trade-off:

> Local caching can improve performance but makes strict global enforcement more difficult.

Therefore hybrid designs need careful quota allocation.

---

# 50. CAP and Consistency Considerations

Distributed rate limiting has a consistency problem.

Suppose a global limit is:

```text
100 requests/minute
```

and there are multiple regions.

If each region has an eventually consistent view:

```text
Region A thinks usage = 40
Region B thinks usage = 40
```

both may allow more requests.

The actual global usage can temporarily exceed:

```text
100
```

If strict enforcement is required, stronger coordination is needed.

If small temporary overshoot is acceptable, a more available and lower-latency design may be chosen.

This is a classic distributed-system trade-off between:

```text
strict consistency
vs
latency/availability
```

---

# 51. Observability

A production rate limiter should expose useful metrics.

Important metrics include:

```text
requests_allowed
requests_rejected
rate_limit_429_count
redis_latency
redis_errors
limiter_latency
bucket_state
hot_keys
```

Useful dimensions:

```text
endpoint
tenant
client
region
service
```

Be careful with extremely high-cardinality labels such as raw user IDs in monitoring systems.

---

## 51.1 Logs

A rejected request may log:

```text
clientId
endpoint
policy
limit
algorithm
remaining
reason
region
```

But sensitive data should be handled according to security/privacy requirements.

---

# 52. Security Considerations

The client identity must be trustworthy.

If the limiter trusts:

```http
X-User-Id: user123
```

from an unauthenticated client, the client may simply change it.

Instead, identity should come from a trusted source such as:

```text
validated authentication token
API key
gateway-authenticated identity
```

For IP-based limits, proxy headers such as:

```text
X-Forwarded-For
```

must only be trusted from configured trusted proxies.

---

# 53. LLD Direction

Once the algorithms are understood, we can build a clean LLD.

A natural abstraction is:

```csharp
public interface IRateLimiter
{
    bool AllowRequest(string clientId);
}
```

Different algorithms can implement it:

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

This naturally leads to the:

> Strategy Pattern.

---

# 53.1 Strategy Pattern

The algorithm is interchangeable.

```csharp
public interface IRateLimiter
{
    bool AllowRequest(string clientId);
}
```

Then:

```csharp
public class FixedWindowRateLimiter : IRateLimiter
{
    public bool AllowRequest(string clientId)
    {
        // Fixed window logic
    }
}
```

and:

```csharp
public class TokenBucketRateLimiter : IRateLimiter
{
    public bool AllowRequest(string clientId)
    {
        // Token bucket logic
    }
}
```

The calling code does not need to know the internal algorithm.

---

# 53.2 Factory

A factory can select the algorithm:

```csharp
public enum RateLimitAlgorithm
{
    FixedWindow,
    SlidingWindowLog,
    SlidingWindowCounter,
    TokenBucket
}
```

Factory:

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

In a real implementation, dependencies/configuration would be injected rather than constructed directly inside the factory.

---

# 53.3 Result object

A production LLD often needs more than:

```text
true / false
```

Instead:

```csharp
public class RateLimitResult
{
    public bool Allowed { get; init; }

    public int? Remaining { get; init; }

    public TimeSpan? RetryAfter { get; init; }

    public int Limit { get; init; }
}
```

Then:

```text
Allowed
Remaining
RetryAfter
Limit
```

can be returned to the gateway/API layer.

---

# 54. Interview Decision Guide

When the interviewer asks:

> Which rate-limiting algorithm would you use?

Do not immediately answer with a name.

First ask:

```text
What behavior do we need?
```

Important questions:

1. Is burst traffic allowed?
2. Do we need a strict rolling window?
3. Is approximate behavior acceptable?
4. How many clients exist?
5. How many requests/sec?
6. Is this single-server or distributed?
7. Do we need a global limit?
8. Is Redis available?
9. What happens if Redis is unavailable?
10. Are multiple regions involved?
11. What latency overhead is acceptable?

---

# 54.1 Choosing Fixed Window

Good when:

```text
simple
cheap
easy to distribute
```

is more important than precise smoothing.

---

# 54.2 Choosing Sliding Window Log

Good when:

```text
precise rolling-window semantics
```

matter and traffic volume is manageable.

---

# 54.3 Choosing Sliding Window Counter

Good when:

```text
rolling behavior
+
low memory
+
approximation is acceptable
```

---

# 54.4 Choosing Token Bucket

Good when:

```text
controlled burst
+
sustained rate
```

are desired.

Example:

```text
capacity = 100
refill = 10/sec
```

This allows a burst of up to:

```text
100 requests
```

while sustaining approximately:

```text
10 requests/sec
```

over time.

---

# 55. Interview-Ready Explanation

A concise but technically strong answer:

> Rate limiting controls how many requests a client can make over a defined policy. I would typically enforce it at the API gateway and possibly again at the service level for expensive operations.
>
> For a single instance, we can keep rate-limit state in memory, but in a horizontally scaled system local state is insufficient because requests can hit different instances. I would use a shared store such as Redis.
>
> The algorithm depends on the required behavior. Fixed Window is simple and O(1), but has a boundary burst problem. Sliding Window Log is more accurate but stores individual timestamps. Sliding Window Counter stores only previous and current counts and provides an approximation. Token Bucket is useful when controlled bursts are desirable because capacity determines burst size and refill rate determines the sustained rate.
>
> For a distributed Token Bucket, Redis can store `tokens` and `lastRefill`. The refill, availability check, token consumption, and state update must be atomic, so I would use a Redis Lua script. Otherwise two API instances could read the same token count and both consume it.
>
> For time, `DateTimeOffset.UtcNow` provides a common UTC reference, but different machines can still have small clock skew. If one authoritative time source is desired, the Lua script can use Redis `TIME`.
>
> I would also consider TTL-based cleanup, Redis failure behavior, hot keys, observability, rate-limit headers, and multi-region consistency.

---

# 56. Common Interview Questions

## Q1. Why can't we keep the counter in API memory?

Because in a distributed deployment:

```text
API1
API2
API3
```

each instance would have a different counter.

A shared store is required for a global policy.

---

## Q2. Why Redis?

Because Redis provides:

- very low latency
- atomic operations
- `INCR`
- TTL
- Lua scripting
- distributed shared state

---

## Q3. Why Lua?

Because:

```text
read
+
modify
+
write
```

must happen atomically.

---

## Q4. Why is `ConcurrentDictionary` not enough?

Because the business operation may contain:

```text
read
check
increment
```

and that entire sequence must be atomic.

---

## Q5. What is the fixed-window boundary problem?

Two consecutive fixed windows can each accept their full limit near the boundary, causing a large burst over a very short period.

---

## Q6. Difference between Sliding Log and Sliding Counter?

Sliding Log:

```text
stores every request timestamp
```

Sliding Counter:

```text
stores counts
```

Therefore:

```text
Log -> more accurate, more memory
Counter -> approximate, less memory
```

---

## Q7. Difference between Token Bucket and Fixed Window?

Fixed Window:

```text
N requests per fixed interval
```

Token Bucket:

```text
tokens accumulate
capacity controls burst
refill rate controls sustained traffic
```

---

## Q8. Can Token Bucket allow a burst?

Yes.

If:

```text
capacity = 10
```

and the bucket is full, up to:

```text
10 requests
```

can be accepted immediately.

---

## Q9. What controls sustained traffic in Token Bucket?

The:

```text
refill rate
```

---

## Q10. Why use `double` for tokens?

Because fractional tokens can accumulate between requests.

---

## Q11. Why use UTC?

To avoid differences caused by local server time zones.

---

## Q12. Does UTC guarantee identical server clocks?

No.

UTC gives a common time reference, but machine clocks can differ slightly.

That difference is clock skew.

---

## Q13. How can Redis provide one central time?

Using:

```lua
redis.call('TIME')
```

---

## Q14. What happens if Redis is down?

Choose explicitly between:

```text
fail-open
```

and:

```text
fail-closed
```

based on the endpoint's availability and protection requirements.

---

## Q15. How do you prevent Redis keys from growing forever?

Use:

```text
TTL
+
expiration
+
cleanup
```

---

## Q16. What is a hot key?

A disproportionately popular Redis key that receives very high traffic.

---

## Q17. How do you scale rate limiting?

Potential techniques:

```text
Redis Cluster
sharding
local pre-filtering
hierarchical limits
quota allocation
regional limits
```

Each has consistency and complexity trade-offs.

---

# 57. Final Cheat Sheet

## Fundamental formula

```text
Rate Limiting
=
Control request rate
```

---

## Fixed Window

```text
counter + window
```

Pros:

```text
simple
cheap
O(1)
```

Cons:

```text
boundary burst
```

---

## Sliding Window Log

```text
timestamps
```

Pros:

```text
accurate
```

Cons:

```text
memory heavy
```

---

## Sliding Window Counter

```text
previous count
+
current count
```

Pros:

```text
low memory
rolling approximation
```

Cons:

```text
not exact
```

---

## Token Bucket

```text
tokens
+
refill rate
+
capacity
```

Pros:

```text
controlled bursts
low memory
O(1)
excellent general-purpose model
```

Cons:

```text
distributed implementation needs atomic state update
```

---

## Redis Fixed Window

```text
INCR
+
EXPIRE
```

Better:

```text
Lua:
INCR
+
EXPIRE
```

---

## Distributed Token Bucket

State:

```text
tokens
lastRefill
```

Algorithm:

```text
elapsed
=
now - lastRefill

tokens
=
min(
    capacity,
    tokens + elapsed * refillRate
)

if tokens >= 1:
    tokens--
    allow
else:
    reject
```

Distributed requirement:

```text
entire operation must be atomic
```

Solution:

```text
Redis Lua
```

---

## Time

```csharp
DateTimeOffset.UtcNow
```

means:

```text
current UTC time
```

```csharp
DateTimeOffset.UtcNow - previousTime
```

means:

```text
elapsed duration
```

```csharp
elapsed.TotalSeconds
```

means:

```text
elapsed seconds
```

```csharp
DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
```

means:

```text
Unix timestamp in milliseconds
```

Redis:

```lua
redis.call('TIME')
```

provides a central Redis time source.

---

# Final Mental Model

If you remember only one picture, remember this:

```text
                        CLIENT
                           |
                           v
                    +-------------+
                    | API Gateway |
                    +------+------+
                           |
                           v
                  +------------------+
                  | Rate Limiter     |
                  |                  |
                  | Identify client  |
                  | Find policy      |
                  | Check state      |
                  | Update state     |
                  +--------+---------+
                           |
                    +------+------+
                    |             |
                 ALLOW          REJECT
                    |             |
                    v             v
                 SERVICE         429
                    |
                    v
               DOWNSTREAM


Distributed state:

API1 --------\
API2 ---------+----> REDIS
API3 --------/


Token Bucket:

              refillRate
                  |
                  v
            +-----------+
            |  TOKENS   |
            |  capacity |
            +-----+-----+
                  |
             1 token/request
                  |
                  v
               ALLOW


Distributed Token Bucket:

API
 |
 v
Redis Lua
 |
 +--> read tokens
 +--> read lastRefill
 +--> get time
 +--> calculate elapsed
 +--> refill
 +--> cap at capacity
 +--> check >= 1
 +--> consume
 +--> save
 +--> return decision


Time:

UTC
 |
 +--> solves time-zone differences
 |
 +--> does NOT guarantee identical physical clocks
 |
 +--> clock skew can still exist
 |
 +--> Redis TIME can provide one central time source
```

---

# End Goal for HLD/LLD

For an interview, the strongest progression is:

```text
1. Explain why rate limiting exists
          |
          v
2. Explain where it sits
          |
          v
3. Choose the rate-limit key
          |
          v
4. Choose algorithm based on requirements
          |
          v
5. Explain local implementation
          |
          v
6. Identify concurrency problems
          |
          v
7. Move to distributed architecture
          |
          v
8. Introduce Redis
          |
          v
9. Make state transitions atomic
          |
          v
10. Discuss failure handling
          |
          v
11. Discuss scaling / hot keys / regions
          |
          v
12. Discuss observability/security
          |
          v
13. Convert into LLD:
       IRateLimiter
       Strategy
       Factory
       Configuration
       Thread safety
       Tests
```

This sequence demonstrates not only knowledge of rate-limiting algorithms, but also the ability to move from a simple local implementation to a production distributed-system design.
