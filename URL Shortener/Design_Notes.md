# Bloomberg System Design Preparation — URL Shortener

## 1. Problem: What is a URL Shortener?

A URL shortener converts a long URL into a short URL.

Example:

```text
Long URL:
https://www.example.com/products/electronics/mobile-phones/product-details?id=12345

Short URL:
https://short.ly/aB91x
```

When a user opens the short URL, the system finds the original URL and redirects the user.

### Core operations

A URL shortener fundamentally has two operations:

1. **Create a short URL**
2. **Redirect a short URL to the original URL**

High-level flow:

```text
CREATE

Client
  |
  | POST long URL
  v
URL Shortener API
  |
  v
Store mapping
  |
  v
Return short URL
```

```text
REDIRECT

User
  |
  | GET /aB91x
  v
URL Shortener API
  |
  v
Find original URL
  |
  v
HTTP Redirect
  |
  v
Original website
```

---

# 2. Why do we need URL Shorteners?

Long URLs can be inconvenient to:

- Share
- Display
- Put in SMS/messages
- Put in QR codes
- Use in marketing campaigns
- Put on social media
- Track through a single identifier

The short URL acts as an identifier for the original URL.

Example:

```text
Original URL
       |
       v
https://example.com/very/long/path/...
       |
       v
Short code
       |
       v
abc123
       |
       v
https://short.ly/abc123
```

The short code itself does not need to contain the entire original URL.

---

# 3. The Fundamental Data Model

At the simplest level we need a mapping:

```text
Short Code / ID  ->  Original URL
```

For example:

```text
ID       Original URL
--------------------------------------
1        https://google.com
2        https://amazon.com
3        https://bloomberg.com
125      https://example.com
```

The database is the source of truth.

---

# 4. Base62

Base62 is one common way of converting a numeric ID into a short string.

## 4.1 Base62 alphabet

We use:

```text
0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
```

There are:

```text
10 digits
26 lowercase letters
26 uppercase letters
--------------------
62 characters
```

Therefore the name is **Base62**.

Mapping:

```text
0-9  -> 0-9

a-z  -> 10-35

A-Z  -> 36-61
```

For example:

```text
0 -> 0
1 -> 1
...
9 -> 9
10 -> a
11 -> b
...
35 -> z
36 -> A
...
61 -> Z
```

---

# 5. Base62 Is NOT Compression

This is important in interviews.

Base62 is an **encoding/representation**, not compression.

Suppose:

```text
125
```

Base62 represents it as:

```text
21
```

The information has not been compressed in the traditional sense.

We are simply representing the same numeric value using a different number system.

Decimal:

```text
125
```

Base62:

```text
21
```

Verification:

```text
2 * 62 + 1
= 124 + 1
= 125
```

---

# 6. Base62 Encoding Algorithm

Suppose:

```text
number = 125
```

Divide by 62 repeatedly.

```text
125 / 62 = 2 remainder 1

2 / 62 = 0 remainder 2
```

The remainders are:

```text
1
2
```

Reverse them:

```text
21
```

Therefore:

```text
125 decimal = 21 Base62
```

## Another example

Encode:

```text
3844
```

```text
3844 / 62 = 62 remainder 0

62 / 62 = 1 remainder 0

1 / 62 = 0 remainder 1
```

Reverse:

```text
100
```

Verify:

```text
1 * 62² + 0 * 62 + 0
= 3844
```

---

# 7. Base62 Encoding Pseudocode

```text
Encode(number):

    if number == 0:
        return "0"

    result = ""

    while number > 0:

        remainder = number % 62

        result += alphabet[remainder]

        number = number / 62

    reverse(result)

    return result
```

The important idea is:

```text
remainder = number % 62
```

The remainder tells us which Base62 character to use.

---

# 8. Base62 C# Implementation

```csharp
using System;
using System.Text;

public static class Base62
{
    private const string Characters =
        "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public static string Encode(long number)
    {
        if (number == 0)
            return "0";

        StringBuilder result = new StringBuilder();

        while (number > 0)
        {
            long remainder = number % 62;

            result.Append(Characters[(int)remainder]);

            number /= 62;
        }

        char[] chars = result.ToString().ToCharArray();

        Array.Reverse(chars);

        return new string(chars);
    }

    public static long Decode(string value)
    {
        long number = 0;

        foreach (char c in value)
        {
            int digit = Characters.IndexOf(c);

            if (digit == -1)
            {
                throw new ArgumentException(
                    $"Invalid Base62 character: {c}");
            }

            number = number * 62 + digit;
        }

        return number;
    }
}
```

---

# 9. Base62 Decoding

Encoding:

```text
125 -> "21"
```

Decoding must perform:

```text
"21" -> 125
```

Formula:

```text
number = number * 62 + digit
```

For `"21"`:

```text
Start:
number = 0

Read '2':
number = 0 * 62 + 2
       = 2

Read '1':
number = 2 * 62 + 1
       = 125
```

Therefore:

```text
"21" -> 125
```

This is important because the redirect request contains the string:

```text
/21
```

and the database lookup can use:

```text
ID = 125
```

---

# 10. Base62 vs Base64

Base62:

```text
0-9
a-z
A-Z
```

Base64 commonly uses:

```text
A-Z
a-z
0-9
+
/
```

and may use:

```text
=
```

as padding.

For URL shorteners, Base62 is convenient because it avoids characters such as:

```text
+
/
=
```

which can have special meaning in URLs or require encoding.

Base62 is therefore a very natural representation for URL short codes.

---

# 11. Is Base62 Mandatory?

No.

A URL shortener can use:

- Base62
- Random strings
- Hashes
- UUID-derived values
- Snowflake-style IDs
- Other unique ID-generation mechanisms

Base62 is simply a compact representation of a number.

A very important interview statement:

> Base62 does not generate uniqueness by itself. It represents an already-generated numeric ID.

---

# 12. Database-Generated ID

A simple architecture is to let SQL Server generate a unique numeric ID.

Example:

```text
UrlMappings

ID       OriginalUrl
--------------------------------
1        https://google.com
2        https://amazon.com
3        https://bloomberg.com
125      https://example.com
```

The process is:

```text
1. Insert URL
2. Database generates ID
3. Application receives generated ID
4. Application converts ID to Base62
5. Application creates short URL
```

Example:

```text
DB ID:
125

Base62:
21

Short URL:
https://short.ly/21
```

Important distinction:

```text
125 != "21"
```

They represent the same logical identifier in different representations.

---

# 13. EF Core Entity

```csharp
public class UrlMapping
{
    public long Id { get; set; }

    public string OriginalUrl { get; set; } = string.Empty;
}
```

---

# 14. DbContext

```csharp
public class UrlShortenerDbContext : DbContext
{
    public UrlShortenerDbContext(
        DbContextOptions<UrlShortenerDbContext> options)
        : base(options)
    {
    }

    public DbSet<UrlMapping> UrlMappings => Set<UrlMapping>();
}
```

Registration:

```csharp
builder.Services.AddDbContext<UrlShortenerDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")));
```

---

# 15. Creating a Short URL

Basic controller:

```csharp
[ApiController]
[Route("api/urls")]
public class UrlsController : ControllerBase
{
    private readonly UrlShortenerDbContext _db;

    public UrlsController(UrlShortenerDbContext db)
    {
        _db = db;
    }

    [HttpPost]
    public async Task<IActionResult> Create(string url)
    {
        var mapping = new UrlMapping
        {
            OriginalUrl = url
        };

        _db.UrlMappings.Add(mapping);

        await _db.SaveChangesAsync();

        long id = mapping.Id;

        string code = Base62.Encode(id);

        string shortUrl =
            $"https://short.ly/{code}";

        return Ok(new { shortUrl });
    }
}
```

---

# 16. What Happens During SaveChangesAsync?

Before:

```csharp
var mapping = new UrlMapping
{
    OriginalUrl = url
};
```

Conceptually:

```text
Id = 0
OriginalUrl = https://example.com
```

Then:

```csharp
await _db.SaveChangesAsync();
```

SQL Server generates:

```text
Id = 125
```

EF Core receives the generated ID and updates the entity:

```text
mapping.Id = 125
```

Then:

```csharp
Base62.Encode(125)
```

returns:

```text
21
```

Final response:

```text
https://short.ly/21
```

---

# 17. Redirect Operation

When the user opens:

```text
https://short.ly/21
```

the system needs to:

```text
21
 |
 | Base62 Decode
 v
125
 |
 | DB lookup
 v
https://example.com
 |
 | HTTP Redirect
 v
Browser opens example.com
```

Controller:

```csharp
[HttpGet("/{code}")]
public async Task<IActionResult> RedirectToOriginal(
    string code)
{
    long id = Base62.Decode(code);

    var mapping = await _db.UrlMappings
        .FindAsync(id);

    if (mapping == null)
        return NotFound();

    return Redirect(mapping.OriginalUrl);
}
```

---

# 18. HTTP Redirect

The application can return a redirect response.

Conceptually:

```http
HTTP/1.1 302 Found
Location: https://google.com
```

The browser sees the `Location` header and requests the destination.

---

# 19. Basic Architecture

At the beginning:

```text
Client
  |
  v
ASP.NET Core API
  |
  v
EF Core
  |
  v
SQL Server
```

This is enough for a basic implementation.

But Bloomberg-level system design asks:

> What happens when traffic becomes extremely large?

That leads to scalability.

---

# 20. Read-Heavy Nature of URL Shorteners

URL shorteners are generally **read-heavy**.

Write:

```text
User creates a short URL
```

Read:

```text
Someone clicks the short URL
```

One short URL can be clicked thousands or millions of times.

Therefore:

```text
Writes << Reads
```

Illustrative example:

```text
10,000 URL creations/sec
1,000,000 redirect requests/sec
```

The exact numbers are assumptions for capacity planning, not universal facts.

The key design characteristic is:

> Redirect traffic can be much larger than URL-creation traffic.

---

# 21. Caching

Because redirects are read-heavy, caching is extremely important.

Instead of querying the database for every redirect:

```text
GET /21
    |
    v
Database
```

we can use:

```text
GET /21
    |
    v
Cache
```

If the URL is already cached, the database is not touched.

---

# 22. Common Cache Types

## 22.1 Client-side cache

Cache exists in:

- Browser
- Device
- Client application

Useful for content that can safely be reused locally.

---

## 22.2 CDN cache

A CDN caches content close to users.

Examples include:

- Cloudflare
- Akamai
- Amazon CloudFront

CDNs are particularly useful for cacheable static or edge-served content.

---

## 22.3 Application in-memory cache

In .NET:

```csharp
IMemoryCache
```

Example:

```text
API #1
  |
Local memory cache
```

Problem in a multi-instance system:

```text
API #1 -> Cache A
API #2 -> Cache B
API #3 -> Cache C
```

The caches are independent.

If API #1 has a cached URL, API #2 may not.

---

## 22.4 Distributed cache

A distributed cache is shared by multiple application instances.

Redis is a common choice.

```text
API #1 \
API #2  ---> Redis
API #3 /
```

All instances can access the same cached data.

---

# 23. Cache-Aside Pattern

We use the **cache-aside** pattern.

Flow:

```text
Application
    |
    v
Check cache
    |
    +---- HIT ----> return cached value
    |
    +---- MISS ---> database
                       |
                       v
                  store in cache
                       |
                       v
                  return value
```

---

# 24. Redis for URL Shortener

A natural Redis structure is:

```text
Key     Value
-------------------------------
21      https://google.com
22      https://amazon.com
23      https://bloomberg.com
```

The cache key is the short code.

This is better than caching:

```text
21 -> 125
```

because then we would still need a database lookup to convert:

```text
125 -> original URL
```

Instead cache:

```text
21 -> original URL
```

Then a cache hit can immediately redirect.

---

# 25. Cache Hit

Request:

```text
GET /21
```

Flow:

```text
User
 |
 v
Load Balancer
 |
 v
.NET API
 |
 v
Redis
 |
 | HIT
 v
https://google.com
 |
 v
302 Redirect
```

No database query is required.

---

# 26. Cache Miss

Request:

```text
GET /21
```

Flow:

```text
User
 |
 v
.NET API
 |
 v
Redis
 |
 | MISS
 v
Database
 |
 v
Original URL
 |
 v
Redis SET
 |
 v
302 Redirect
```

This means the next request can be served from Redis.

---

# 27. Conceptual C# Cache-Aside Code

```csharp
var originalUrl =
    await redis.GetStringAsync(code);

if (originalUrl == null)
{
    var mapping = await db.UrlMappings
        .FirstOrDefaultAsync(x => x.Id == id);

    if (mapping == null)
        return NotFound();

    originalUrl = mapping.OriginalUrl;

    await redis.SetStringAsync(
        code,
        originalUrl);
}

return Redirect(originalUrl);
```

---

# 28. Cache TTL

Cache entries normally have a TTL:

```text
TTL = Time To Live
```

Example:

```text
Key:
21

Value:
https://google.com

TTL:
1 hour
```

After one hour:

```text
Redis entry expires
```

But:

```text
Database entry remains
```

because the database is the source of truth.

Next request:

```text
Redis MISS
   |
   v
Database
   |
   v
Redis SET
```

---

# 29. Cache Invalidation

Suppose the database changes:

```text
ID 125
old URL -> https://oldsite.com
```

to:

```text
ID 125
new URL -> https://newsite.com
```

But Redis still contains:

```text
21 -> https://oldsite.com
```

Then users may temporarily receive the old URL.

A common approach:

```text
Update DB
   |
   v
Delete Redis key
   |
   v
Next request gets cache miss
   |
   v
Load new value from DB
   |
   v
Populate Redis
```

This is cache invalidation.

---

# 30. Redis Failure

A simplistic fallback is:

```text
Redis unavailable
       |
       v
Read from database
```

But at extremely high traffic this can be dangerous.

For example:

```text
1,000,000 requests/sec
       |
       v
Redis fails
       |
       v
1,000,000 DB requests/sec
```

The database can become overwhelmed.

Therefore a high-scale system needs:

- Redis redundancy
- Failover
- Capacity planning
- Controlled degradation
- Database protection

---

# 31. Horizontal Scaling of the API

One API instance cannot necessarily handle unlimited traffic.

Instead:

```text
             Users
               |
               v
         Load Balancer
          /     |     \
         v      v      v
      API #1 API #2 API #3
          \     |     /
               v
             Redis
               |
               v
              DB
```

This is **horizontal scaling**.

Horizontal scaling means:

> Add more instances of the service.

---

# 32. Vertical Scaling

Vertical scaling means making one machine more powerful.

For example:

```text
8 CPU
32 GB RAM
```

becomes:

```text
32 CPU
128 GB RAM
```

It is simpler but has physical/cost limits.

Horizontal scaling generally provides more flexibility for stateless application services.

---

# 33. Why the API Can Be Stateless

Suppose:

```text
GET /21
```

API #1 handles it.

Another request:

```text
GET /21
```

can be handled by API #3.

The API does not need local session state to understand the URL.

The shared state lives in:

```text
Redis
Database
```

Therefore:

```text
API #1
API #2
API #3
```

can all behave the same way.

This makes horizontal scaling easier.

---

# 34. Load Balancer

The load balancer distributes incoming requests.

```text
             Users
               |
               v
        +--------------+
        | Load Balancer|
        +--------------+
          /    |    \
         /     |     \
        v      v      v
      API1   API2   API3
```

It can also perform health checks.

If API #2 is unhealthy:

```text
API #2 -> unhealthy
```

the load balancer can stop sending new traffic to it.

---

# 35. Sticky Sessions

Sticky sessions mean a user is repeatedly routed to the same API instance.

For the basic URL-shortener architecture, sticky sessions are generally unnecessary because the API is stateless.

That is desirable because:

```text
Request 1 -> API #1
Request 2 -> API #3
Request 3 -> API #2
```

can all work correctly.

---

# 36. Redis Replication

Redis can use replicas.

Conceptually:

```text
             Redis Primary
              /         \
             v           v
         Replica 1    Replica 2
```

Replication means:

> Multiple copies of the same dataset exist.

Benefits:

- Redundancy
- Failover
- Potential read scaling depending on configuration

Important:

Replication alone does **not** solve dataset-size scaling.

If every node contains the entire dataset:

```text
Primary = entire dataset
Replica = entire dataset
Replica = entire dataset
```

---

# 37. Redis Sharding

Sharding means splitting the dataset across nodes.

Example:

```text
              Redis Cluster
             /      |      \
            v       v       v
         Node 1   Node 2   Node 3
```

Conceptually:

```text
key "21"
   |
   v
hash / partition calculation
   |
   v
Redis Node 2
```

The application does not manually decide:

```text
21 -> Node 2
22 -> Node 3
```

A Redis cluster/partitioning mechanism determines where the key belongs.

---

# 38. Replication vs Sharding

Very important interview distinction.

### Replication

Copies data.

```text
Primary
  |
  +-- Replica
  |
  +-- Replica
```

Primary goal:

```text
Redundancy / availability
```

### Sharding

Splits data.

```text
Node 1 -> part of data
Node 2 -> part of data
Node 3 -> part of data
```

Primary goal:

```text
Scale dataset and workload
```

They can be combined:

```text
Shard 1 -> Primary + Replica
Shard 2 -> Primary + Replica
Shard 3 -> Primary + Replica
```

---

# 39. Database Indexing

For a redirect request:

```text
ID = 125
```

we want:

```sql
SELECT Id, OriginalUrl
FROM UrlMappings
WHERE Id = 125;
```

The ID should be indexed, normally as the primary key.

Without a useful index, the database may need to scan many rows.

With an index:

```text
ID -> quickly locate row
```

This is one of the first database optimizations.

---

# 40. Read Replicas

A database can also have read replicas.

```text
             Primary DB
              /      \
             v        v
        Read Replica  Read Replica
```

Typical model:

```text
Writes -> Primary
Reads  -> Replicas
```

This can scale read traffic.

But:

> Read replicas do not solve primary write scaling.

Also, replicas may have replication lag.

---

# 41. Database Replication

Conceptually:

```text
Primary DB
    |
    | transaction/change log
    v
Replication mechanism
    |
    +------> Replica 1
    |
    +------> Replica 2
```

The primary does not normally resend the entire database for every change.

Changes are captured and applied to replicas.

---

# 42. Synchronous Replication

Conceptually:

```text
Application
    |
    v
Primary
    |
    v
Replica confirms
    |
    v
Application receives success
```

Benefits:

- Stronger consistency/durability characteristics

Costs:

- Higher latency
- Replica availability can affect writes depending on configuration

---

# 43. Asynchronous Replication

Conceptually:

```text
Application
    |
    v
Primary
    |
    v
ACK returned
    |
    +------ later ------> Replica
```

Benefits:

- Lower write latency
- Primary is less coupled to replica timing

Cost:

- Replica can temporarily be behind

This is called **replication lag**.

---

# 44. Read-After-Write Problem

Suppose:

```text
POST /api/urls
```

creates:

```text
ID = 125
Base62 = 21
```

The write goes to primary.

Immediately after that:

```text
GET /21
```

may be routed to a read replica.

If asynchronous replication has not caught up:

```text
Replica does not know ID 125 yet
```

The user could temporarily receive:

```text
404 Not Found
```

even though the write succeeded.

---

# 45. One Solution: Populate Redis During Write

Write flow:

```text
POST
 |
 v
Primary DB
 |
 v
ID = 125
 |
 v
Base62 = 21
 |
 v
Redis SET
 |
 v
Return short URL
 |
 v
Replication happens asynchronously
```

Then:

```text
GET /21
 |
 v
Redis
 |
 v
HIT
```

So the immediate read does not depend on replica propagation.

Other approaches include routing consistency-sensitive reads to the primary.

---

# 46. Current Architecture

At this stage our architecture is:

```text
                         USERS
                           |
                           v
                    LOAD BALANCER
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
           API #1        API #2        API #3
             |             |             |
             +-------------+-------------+
                           |
                           v
                    REDIS CLUSTER
                    /      |      \
                   v       v       v
                Shard    Shard    Shard
                           |
                     Cache Miss
                           |
                           v
                     READ REPLICAS
                           |
                           v
                      PRIMARY DB
                           ^
                           |
                  Async Replication
```

---

# 47. Write Path

The current conceptual write path:

```text
POST /api/urls
        |
        v
Load Balancer
        |
        v
.NET API
        |
        v
Primary Database
        |
        v
Generate numeric ID
        |
        v
Base62 Encode
        |
        v
Store mapping in Redis
        |
        v
Return short URL
```

Example:

```text
Original:
https://example.com/products/123

DB generates:
125

Base62:
21

Response:
https://short.ly/21
```

---

# 48. Read Path

The current conceptual read path:

```text
GET /21
   |
   v
Load Balancer
   |
   v
.NET API
   |
   v
Redis
   |
   +-------- HIT --------> Original URL
   |                         |
   |                         v
   |                       302
   |
   +-------- MISS -------> Read Replica
                             |
                             v
                         Original URL
                             |
                             v
                        Redis SET
                             |
                             v
                            302
```

---

# 49. Important Interview Concepts Already Covered

## Base62

```text
Numeric ID -> compact URL-safe representation
```

## Database ID

```text
Unique internal identifier
```

## Cache

```text
Performance layer
```

## Redis

```text
Distributed cache
```

## Cache-aside

```text
Cache first
DB on miss
Populate cache
```

## Horizontal scaling

```text
Add API instances
```

## Replication

```text
Copy data
```

## Sharding

```text
Split data
```

## Read replicas

```text
Scale reads
```

## Replication lag

```text
Replica temporarily behind primary
```

## Stateless API

```text
Any API instance can handle a request
```

---

# 50. Important Mental Model

Keep this distinction in mind:

```text
ID generation
      |
      v
Unique numeric ID
      |
      v
Base62 encoding
      |
      v
Short code
```

Do NOT think:

```text
Base62 generates uniqueness
```

Instead:

```text
ID generator generates uniqueness
Base62 makes the ID compact
```

---

# 51. Important Scalability Mental Model

Think about the system in layers:

```text
                 TRAFFIC
                    |
                    v
             Load Balancer
                    |
                    v
          Stateless API Layer
                    |
                    v
             Redis / Cache
                    |
             Cache misses
                    |
                    v
            Database Reads
                    |
                    v
            Database Writes
```

At each layer ask:

> What happens when traffic becomes 10x?

That is the essence of system-design thinking.

---

# 52. What We Have NOT Yet Covered

These are the next important topics for the full Bloomberg system-design round.

## 52.1 Unique ID Generation — Deep Dive

Need to cover:

- SQL Identity
- SQL Sequence
- Concurrent inserts
- Multiple API servers
- GUID
- Snowflake-style IDs
- Distributed ID generation
- Why Base62 comes after ID generation

This is the **next recommended topic**.

---

## 52.2 Database Write Scaling

Need to cover:

- Primary DB bottleneck
- Connection pooling
- Batching
- Partitioning
- Database sharding
- Write distribution

---

## 52.3 Database Sharding

Need to understand:

```text
Why shard?
How to shard?
How to choose shard key?
What happens during rebalancing?
What happens when a shard fails?
```

---

## 52.4 CAP Theorem

Need to cover deeply:

```text
Consistency
Availability
Partition Tolerance
```

Then:

```text
CP
AP
```

And specifically understand how CAP relates to the URL-shortener architecture.

Important:

> Replication lag by itself is not CAP.

CAP becomes relevant when reasoning about distributed systems under network partitions.

---

## 52.5 Reliability

Need to cover:

- Failover
- Health checks
- Retry
- Timeout
- Circuit breaker
- Graceful degradation
- Backpressure
- Dependency failures

---

## 52.6 Consistency Decisions

Need to discuss:

- URL immutability
- URL updates
- Deletes
- Expiration
- Duplicate URLs
- Strong consistency
- Eventual consistency

---

## 52.7 Collision and Uniqueness

Need to understand:

- Random short codes
- Sequential IDs
- Unique database constraints
- Collision detection
- Collision retries
- Code length
- Security implications of sequential IDs

---

## 52.8 Capacity Estimation

Need to calculate:

```text
URLs/day
Writes/sec
Reads/sec
Storage
Redis memory
Bandwidth
Database size
```

This is extremely important in system design interviews.

---

## 52.9 Load Balancer Deep Dive

Need to cover:

- L4
- L7
- Round robin
- Least connections
- Health checks
- Sticky sessions
- Failover

---

## 52.10 Rate Limiting

Need to cover:

- Fixed window
- Sliding window
- Token bucket
- Leaky bucket
- Redis-based rate limiting

---

## 52.11 Security

Need to cover:

- HTTPS
- URL validation
- Malicious URLs
- Open redirects
- Abuse prevention
- Authentication for creation APIs
- Authorization
- Rate limiting

---

## 52.12 Observability

Need to cover:

```text
Logs
Metrics
Traces
Alerts
```

Important URL-shortener metrics:

```text
Request rate
P50/P95/P99 latency
Cache hit ratio
Cache miss ratio
DB latency
Redis latency
Replication lag
Error rate
404 rate
Redirect rate
```

---

# 53. How to Explain the Design in an Interview

A strong structure is:

```text
1. Clarify requirements

2. Define APIs

3. Define data model

4. Explain basic architecture

5. Explain write flow

6. Explain read/redirect flow

7. Identify read-heavy nature

8. Add Redis cache

9. Add horizontal API scaling

10. Add Redis scaling

11. Add DB read replicas

12. Discuss consistency

13. Discuss ID generation

14. Discuss database write scaling

15. Discuss failures

16. Capacity estimation

17. Security

18. Observability
```

Do not immediately jump into complicated distributed systems.

Start simple:

```text
API -> DB
```

Then explain why that is insufficient at scale.

Then evolve:

```text
API -> Redis -> DB
```

Then:

```text
Load Balancer
       |
   API fleet
       |
    Redis
       |
   DB replicas
       |
   Primary DB
```

This evolution demonstrates system-design thinking.

---

# 54. Key Interview Statements

### About Base62

> Base62 is a compact representation of a numeric ID using 62 URL-friendly characters. It does not generate uniqueness by itself.

### About Redis

> Redis is a performance layer; the database remains the source of truth.

### About cache-aside

> The application first checks Redis. On a miss it reads from the database, populates Redis, and returns the result.

### About horizontal scaling

> Because the API is stateless, we can run multiple instances behind a load balancer.

### About replication

> Replication copies data for redundancy and potentially read scaling, while sharding distributes different portions of the dataset.

### About read replicas

> Read replicas help scale read traffic but do not solve the primary database's write bottleneck.

### About replication lag

> With asynchronous replication, a read replica may temporarily lag behind the primary, so a read immediately after a write can observe stale or missing data.

---

# 55. Final Mental Picture

The URL shortener can be thought of as:

```text
                 +----------------+
                 |     USERS      |
                 +-------+--------+
                         |
                         v
                 +---------------+
                 | Load Balancer |
                 +-------+-------+
                         |
              +----------+----------+
              |          |          |
              v          v          v
            API #1     API #2     API #3
              |          |          |
              +----------+----------+
                         |
                         v
                +------------------+
                | Redis Cluster    |
                | Cache / Shards   |
                +--------+---------+
                         |
                     Cache Miss
                         |
                         v
                +------------------+
                | Read Replicas    |
                +--------+---------+
                         |
                         v
                +------------------+
                | Primary Database |
                +------------------+
                         ^
                         |
                 Replication
```

The core request flow is:

```text
CREATE

Long URL
   |
   v
API
   |
   v
Primary DB
   |
   v
Unique ID
   |
   v
Base62
   |
   v
Short Code
   |
   v
Redis
   |
   v
Response
```

And:

```text
REDIRECT

Short Code
   |
   v
API
   |
   v
Redis
   |
   +---- HIT ----> Original URL ----> 302
   |
   +---- MISS ---> DB
                    |
                    v
                Redis SET
                    |
                    v
                   302
```

---

# 56. One-Line Summary of the Entire Design So Far

> Build a stateless ASP.NET Core URL-shortener service that generates a unique numeric ID, represents it using Base62, stores the URL mapping in a primary database, uses Redis as a distributed cache for read-heavy redirects, horizontally scales API instances behind a load balancer, and uses database/Redis replication and sharding where required for scale and availability.

---

# 57. Next Study Topic

Continue from:

# Unique ID Generation

Start from absolute basics:

```text
Why do we need an ID?
        |
        v
Can SQL Identity solve it?
        |
        v
What happens with 10 API servers?
        |
        v
Can every server safely generate IDs?
        |
        v
What is SQL Sequence?
        |
        v
Why GUID?
        |
        v
Why Snowflake?
        |
        v
Distributed ID generation
        |
        v
Then Base62
```

This is the next major part of the URL-shortener system design.
