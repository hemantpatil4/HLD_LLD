# Bloomberg URL Shortener --- Complete Caching & Redis Deep Dive

> Interview-preparation chapter: caching from absolute basics to
> production architecture.

## 1. What is a Cache?

A cache is a faster storage layer that keeps a copy of data that is
expensive or slow to retrieve.

Without cache:

``` text
Client -> API -> Database -> Response
```

With cache:

``` text
Client -> API -> Cache
                  |  \
                 HIT MISS
                  |    \
               Return  DB
                       |
                     Cache
                       |
                    Return
```

Core principle:

> Cache is normally a performance layer. The database remains the source
> of truth.

## 2. Why do we need caching?

Suppose the URL shortener receives 1,000,000 redirect requests/sec. If
every request reaches the database, the database becomes the bottleneck.

With a 99% cache hit ratio:

``` text
1,000,000 redirects/sec
          |
        Redis
       /     \
   99% HIT   1% MISS
     |          |
  990K/sec     10K/sec
                |
                DB
```

Caching reduces latency and protects expensive downstream systems.

## 3. What makes good cache data?

Good candidates are:

-   Frequently read
-   Expensive to retrieve or compute
-   Relatively stable
-   Reusable by many requests

For our URL shortener:

``` text
url:aZ91k -> https://www.google.com
```

is an excellent candidate because one short URL can receive millions of
redirects.

## 4. Cache vs Database

### Database

Provides durable storage, transactions, constraints, querying and
recovery mechanisms.

### Cache

Optimized for very fast access, high throughput and temporary copies.

``` text
             Source of truth
                   |
                   v
                Database
                   ^
                   |
                 Cache
                   ^
                   |
                  API
```

Do not automatically treat a cache as the durable source of truth.

# 5. Major types of cache

There are multiple caching layers:

``` text
Client / Browser
       |
       v
CDN / Edge
       |
       v
Application Local Memory
       |
       v
Distributed Cache (Redis)
       |
       v
Database
```

The most important application distinction is:

``` text
Local cache vs Distributed cache
```

## 5.1 Browser / client cache

The client can cache HTTP responses using HTTP caching semantics such as
`Cache-Control`.

Advantages: - Lowest possible latency - No backend request - Reduces
server traffic

Disadvantages: - Less server-side control after response delivery -
Potentially stale content depending on cache policy

For a public URL redirect service, redirect caching behavior should be
designed deliberately.

## 5.2 CDN / edge cache

A CDN caches data near users:

``` text
Internet
   |
 CDN / Edge
 /   |   \
E1   E2   E3
 \   |   /
   Origin
      |
     API
```

Benefits: - Geographic latency reduction - Absorbs repeated public
traffic - Protects origin - Excellent for extremely popular public
content

## 5.3 Application in-memory cache

.NET provides `IMemoryCache`.

``` csharp
builder.Services.AddMemoryCache();
```

Example:

``` csharp
if (!_cache.TryGetValue(code, out string? originalUrl))
{
    originalUrl = await LoadFromDatabase(code);
    _cache.Set(code, originalUrl, TimeSpan.FromMinutes(10));
}

return Redirect(originalUrl);
```

The important property is that each API instance has its own cache.

## 5.4 Distributed cache

``` text
API1 \
API2  +--> Redis --> DB
API3 /
```

All API instances share the same cache.

Benefits: - Shared across horizontally scaled APIs - Centralized cache
population - Individual API restart does not necessarily empty the whole
cache

Trade-off: additional network hop and operational complexity.

## 5.5 Multi-level cache

A production system can combine caches:

``` text
Request
  |
  v
L1: Local Memory
  |
 MISS
  v
L2: Redis
  |
 MISS
  v
L3: Database
```

On an L2 hit, populate L1. On an L3 hit, populate both L2 and L1.

This can provide extremely low latency, but invalidation becomes more
complex because multiple copies exist.

# 6. Local vs Distributed Cache

  Property             Local memory     Distributed cache
  -------------------- ---------------- -------------------------
  Location             API process      Separate cache system
  Shared across APIs   No               Yes
  Latency              Lowest           Very low
  Restart              Cache lost       Can survive API restart
  Horizontal scaling   More difficult   Natural
  Invalidation         Per instance     Centralized/shared
  Complexity           Low              Higher

# 7. What should we cache?

For the URL shortener, cache the complete redirect target:

``` text
Key:   url:aZ91k
Value: https://www.google.com
```

Do not normally cache only:

``` text
aZ91k -> 125
```

because then the application still needs a database lookup for ID 125.

The ideal cache hit is:

``` text
shortCode -> originalUrl -> HTTP 302
```

# 8. Cache key design

Good cache keys are deterministic, unique and namespaced.

``` text
url:aZ91k
user:123
clicks:aZ91k
```

Namespacing prevents collisions and makes operational debugging easier.

# 9. Cache-aside / Lazy Loading

This is the primary cache pattern for our URL shortener.

``` text
Application
    |
 Check cache
   /     \
 HIT     MISS
  |         |
Return      DB
             |
             v
           Cache
             |
           Return
```

The application owns the cache logic.

### Read flow

``` text
GET /aZ91k
    |
    v
Redis GET url:aZ91k
   /           \
 HIT           MISS
  |              |
  v              v
302             DB
                 |
                 v
             Redis SET
                 |
                 v
                302
```

### C# example

``` csharp
var key = $"url:{code}";
var originalUrl = await redis.StringGetAsync(key);

if (originalUrl.IsNullOrEmpty)
{
    long id = Base62.Decode(code);

    var mapping = await db.UrlMappings
        .AsNoTracking()
        .FirstOrDefaultAsync(x => x.Id == id);

    if (mapping == null)
        return NotFound();

    originalUrl = mapping.OriginalUrl;

    await redis.StringSetAsync(
        key,
        originalUrl,
        TimeSpan.FromHours(1));
}

return Redirect(originalUrl.ToString());
```

# 10. Write strategies

There are several important write patterns.

## 10.1 Cache-aside write

Application writes the DB and then manages cache invalidation/population
itself.

Typical update:

``` text
DB update
   |
Delete cache key
```

For an immutable URL mapping, creation can populate Redis immediately
after the DB write.

## 10.2 Write-through

``` text
Application -> Cache -> Database
```

The cache participates in the synchronous write path and updates the
backing store.

Advantages: - Cache populated on writes - Future reads fast

Disadvantages: - Higher write-path coupling - Potentially higher write
latency

## 10.3 Write-behind / write-back

``` text
Application -> Cache
                 |
                 | asynchronous
                 v
              Database
```

Very fast writes, but more complex. Data can be lost if the cache fails
before persistence and ordering/recovery become harder.

For URL creation, this is generally not the default because the mapping
should be durably stored before returning success.

## 10.4 Read-through

The application talks to the cache, and the cache itself knows how to
load the backing store on a miss.

Difference:

``` text
Cache-aside: Application handles MISS -> DB -> SET
Read-through: Cache handles MISS -> backing store
```

# 11. Cache invalidation

One of the hardest cache problems is deciding when cached data is no
longer valid.

Common mechanisms:

-   TTL
-   Explicit deletion
-   Versioned keys
-   Write-through
-   Refresh-ahead

## 11.1 TTL --- Time To Live

Example:

``` text
url:aZ91k -> google.com
TTL = 3600 seconds
```

After the TTL expires, the cache copy becomes unavailable. The database
record is unaffected.

TTL is useful as a safety mechanism against stale data.

## 11.2 Explicit invalidation

For a mutable value:

``` text
Update DB
   |
Delete Redis key
```

Next read:

``` text
MISS -> DB -> SET -> return
```

## 11.3 Why use both invalidation and TTL?

Explicit invalidation handles known updates quickly. TTL is a safety net
if invalidation is missed or if some data should only live in cache for
a limited period.

# 12. TTL jitter

If millions of keys are inserted simultaneously with exactly the same
TTL, they can expire simultaneously:

``` text
Millions of keys
      |
      v
same expiry time
      |
      v
huge MISS spike
      |
      v
DB overload
```

Use randomized TTLs:

``` text
TTL = base TTL + small random jitter
```

This spreads expirations over time.

# 13. Refresh-ahead

Instead of waiting for a popular key to expire:

``` text
TTL nearly expired
        |
        v
background refresh
        |
        v
new TTL
```

Useful for expensive, frequently accessed data.

# 14. Cache stampede

A cache stampede happens when many requests simultaneously miss the same
key.

``` text
1,000,000 requests
        |
        v
Redis MISS
        |
        +--> DB
        +--> DB
        +--> DB
        +--> DB
```

This can overload the database even though Redis normally protects it.

## Mitigations

### Single-flight / request coalescing

Only one request loads the key:

``` text
Many requests
     |
   MISS
     |
 single-flight
     |
  one DB call
     |
 Redis SET
     |
 all requests use result
```

### Distributed lock

One request obtains a lock, loads the value and populates cache; other
requests wait/recheck.

Locks need careful timeout, ownership and failure handling.

### Refresh-ahead

Refresh before expiry.

### TTL jitter

Spread expiry times.

# 15. Hot keys

A hot key receives disproportionate traffic.

Example:

``` text
url:aZ91k -> 500,000 requests/sec
other keys -> 10 requests/sec each
```

A single key can become a hotspot even if total cache traffic is
acceptable.

Mitigations:

-   Local L1 cache
-   CDN/edge caching
-   Request coalescing
-   Appropriate replication/distribution
-   Avoid unnecessary repeated backend work

# 16. Cache penetration

Cache penetration occurs when requests repeatedly ask for data that does
not exist:

``` text
Redis MISS -> DB MISS
Redis MISS -> DB MISS
Redis MISS -> DB MISS
```

Examples:

``` text
/random-invalid-code-1
/random-invalid-code-2
...
```

Mitigations:

-   Input validation
-   Negative caching
-   Rate limiting
-   Abuse protection
-   Bloom filters where appropriate

# 17. Negative caching

Cache a known absence for a short TTL:

``` text
url:unknown123 -> NOT_FOUND
TTL = 30 sec
```

Repeated invalid requests are answered from cache instead of repeatedly
hitting the DB.

Keep negative TTL short enough that a newly created key is not hidden
for too long.

# 18. Cache avalanche

A cache avalanche is a broad event where many cached entries disappear
or become unavailable at once:

``` text
Millions of keys expire/fail
          |
          v
Millions of backend requests
          |
          v
Database overload
```

Mitigations:

-   TTL jitter
-   Staggered expiration
-   Highly available cache
-   Multi-level cache
-   Request coalescing
-   DB protection

# 19. Cache breakdown / hot-key expiration

A popular key expires and creates concentrated backend load:

``` text
Hot key expires
      |
      v
Huge concurrent MISS
      |
      v
DB pressure
```

This is narrower than an avalanche because the problem is concentrated
around one/few very hot entries.

# 20. Cache eviction

Cache memory is finite. When memory pressure occurs, entries may be
removed according to the configured policy.

Common concepts:

-   LRU --- Least Recently Used
-   LFU --- Least Frequently Used
-   FIFO --- First In, First Out
-   Random eviction
-   TTL-aware policies

## LRU

Remove the item that has not been used recently.

Useful when recent access predicts future access.

## LFU

Remove items with low access frequency.

Useful when frequently used items should remain cached even if their
last access was not extremely recent.

## FIFO

Remove the oldest inserted entry regardless of later access.

Important: real Redis policies are configurable and some are
approximations/TTL-aware; do not describe Redis as always using textbook
LRU.

# 21. Eviction vs expiration

These are different.

### Expiration

TTL ended:

``` text
TTL -> 0 -> key expires
```

### Eviction

Memory/resource policy removes a key:

``` text
Memory pressure -> policy selects key -> key removed
```

A key can disappear for either reason.

# 22. Redis data structures

Important Redis structures:

``` text
String
Hash
List
Set
Sorted Set
Stream
```

For URL mapping, a String is enough:

``` text
SET url:aZ91k https://google.com
```

## String

Good for: - URL mappings - Counters - Tokens - JSON blobs

## Hash

Useful for object-like values:

``` text
url:aZ91k
  |- originalUrl
  |- ownerId
  |- createdAt
```

## Set

Useful for unique membership, e.g. a user's URL IDs.

## Sorted Set

Useful for ranking, e.g. popular URLs by click count.

## Stream

Useful for event/stream processing use cases such as asynchronous
analytics pipelines.

# 23. Redis counters

For click counts:

``` text
INCR clicks:aZ91k
```

But high-volume analytics should not necessarily block the redirect
path.

Better architecture:

``` text
Redirect
   |
   +--> immediate 302
   |
   +--> event/queue
           |
           v
       analytics
```

# 24. Redis replication

Replication copies data:

``` text
Redis Primary
   |
   +--> Replica 1
   +--> Replica 2
```

Benefits:

-   Redundancy
-   Failover
-   Potential read scaling

Replication is not the same as sharding.

# 25. Redis sharding

Sharding splits keys across nodes:

``` text
Redis Cluster
 /    |    \
S1    S2    S3
```

Example concept:

``` text
hash(url:aZ91k) -> S2
hash(url:bX22m) -> S1
hash(url:cY33n) -> S3
```

Benefits:

-   More total memory
-   More aggregate throughput
-   Horizontal scaling

Costs:

-   Routing
-   Rebalancing
-   Operational complexity
-   Hot-key concerns
-   Multi-key operation constraints depending on key placement

# 26. Replication vs sharding

Replication:

``` text
Primary -> copies
```

Sharding:

``` text
Data -> pieces
```

Can combine:

``` text
Shard 1: Primary + Replicas
Shard 2: Primary + Replicas
Shard 3: Primary + Replicas
```

# 27. Redis and consistent hashing

A basic modulo scheme:

``` text
shard = hash(key) % N
```

If N changes, many keys can move.

Consistent hashing reduces the amount of remapping when nodes change.

In Redis Cluster specifically, hash slots are used rather than generic
application-level consistent hashing. For interview discussion,
distinguish the generic consistent-hashing concept from Redis Cluster's
slot-based routing.

# 28. Redis failure

Normal path:

``` text
API -> Redis -> 302
```

Potential fallback:

``` text
API -> DB -> 302
```

But at 1M RPS, a Redis outage could send 1M requests/sec toward the DB
and cause a cascading failure.

Therefore:

``` text
Redis failure
   |
   +--> timeout
   +--> circuit breaker
   +--> bounded concurrency
   +--> rate limiting
   +--> controlled degradation
```

"Fall back to DB" is not sufficient by itself.

# 29. Connection management

Do not create a new Redis connection for every HTTP request.

In .NET with StackExchange.Redis, the common pattern is to reuse a
`ConnectionMultiplexer`.

Conceptually:

``` csharp
var multiplexer =
    await ConnectionMultiplexer.ConnectAsync(redisConnectionString);

builder.Services.AddSingleton<IConnectionMultiplexer>(multiplexer);
```

Then obtain a logical database:

``` csharp
var db = multiplexer.GetDatabase();
```

Connection sizing should consider total API instance count and Redis
capacity.

# 30. Redis C# examples

### Read

``` csharp
var key = $"url:{code}";
var value = await db.StringGetAsync(key);

if (!value.IsNullOrEmpty)
{
    return Redirect(value.ToString());
}
```

### Write with TTL

``` csharp
await db.StringSetAsync(
    $"url:{code}",
    originalUrl,
    TimeSpan.FromHours(1));
```

### Delete

``` csharp
await db.KeyDeleteAsync($"url:{code}");
```

### Counter

``` csharp
await db.StringIncrementAsync($"clicks:{code}");
```

# 31. Cache service abstraction

Keep cache-specific code behind a service when the application grows:

``` csharp
public interface IUrlCache
{
    Task<string?> GetAsync(string code);
    Task SetAsync(string code, string url, TimeSpan ttl);
    Task RemoveAsync(string code);
}
```

This keeps controller/business logic independent of the Redis API.

# 32. Cache serialization

For simple URL mappings:

``` text
string -> string
```

is preferable to serializing a complex object.

For objects:

``` text
Object -> JSON -> Redis
Redis -> JSON -> Object
```

Serialization introduces CPU and payload overhead, so cache only what
the read path needs.

# 33. Cache key versioning

If cache format changes during deployment:

``` text
url:v1:aZ91k
```

can become:

``` text
url:v2:aZ91k
```

Versioned keys avoid incompatible old/new representations during
migrations.

# 34. Cache warming

After a cache restart:

``` text
Redis = empty
```

Lazy loading rebuilds the cache from traffic:

``` text
MISS -> DB -> SET
```

For known hot data, proactive warming can load selected entries before
traffic arrives.

Do not blindly load an entire huge database into cache.

# 35. Cache warming vs lazy loading

Lazy loading:

``` text
Request -> MISS -> DB -> Cache
```

Simple and demand-driven.

Warming:

``` text
Startup/preparation -> load hot data -> Cache -> traffic
```

Useful for predictable hot data but requires knowing what to warm.

# 36. Cache consistency

Once we cache data, we have two copies:

``` text
DB = B
Redis = A
```

The application must define what stale data is acceptable.

For mutable data, a common pattern is:

``` text
UPDATE DB
    |
DELETE Redis
```

Then:

``` text
READ -> MISS -> DB -> SET
```

# 37. Update cache vs delete cache

Suppose:

``` text
DB = A
Cache = A
```

DB changes to B.

Option 1:

``` text
SET Cache = B
```

Option 2:

``` text
DELETE Cache
```

Deletion is often simpler because the next read naturally obtains the
source-of-truth value.

However, concurrent readers/writers can create race conditions, so
frequently changing data may require versioning or stronger
coordination.

# 38. Race condition during invalidation

Suppose:

``` text
DB = A
Cache = A
```

Writer wants B while a reader is loading A.

A possible sequence:

``` text
Reader -> cache MISS
Writer -> DB = B
Writer -> DELETE cache
Reader -> DB read returns A due to timing/replica lag
Reader -> SET cache = A
```

Now stale A has been repopulated after invalidation.

This is why cache consistency can require ordering, version checks,
primary reads, locks or other coordination for mutable data.

For the URL shortener, immutable mappings dramatically reduce this
problem.

# 39. Why immutable URL mappings are ideal for caching

If:

``` text
aZ91k -> https://google.com
```

never changes, we do not have a normal update/invalidation race.

Creation can simply do:

``` text
DB commit -> Redis SET -> return
```

The main cache invalidation concern becomes expiration/eviction rather
than frequent data updates.

# 40. Read replica vs cache

These solve different problems.

Cache:

``` text
Repeated hot reads -> Redis
```

Goal:

``` text
Lower latency + reduce DB load
```

Read replica:

``` text
DB reads -> replica
```

Goal:

``` text
Scale database reads
```

They can be combined:

``` text
API -> Redis
       |
      MISS
       |
       v
Read Replica
```

# 41. Cache hit ratio

Formula:

``` text
Hit Ratio = Hits / (Hits + Misses)
```

Example:

``` text
Hits   = 990,000
Misses = 10,000

Hit Ratio = 99%
```

The miss rate is 1%.

A falling hit ratio can dramatically increase DB traffic.

# 42. Capacity example

Suppose:

``` text
Redirects = 1,000,000/sec
Hit ratio = 99%
```

Then approximately:

``` text
Redis traffic = 1,000,000/sec
DB reads      = 10,000/sec
```

If hit ratio falls to 80%:

``` text
DB reads = 200,000/sec
```

That is a 20x increase in DB reads compared with the 99% hit-ratio case.

Therefore cache hit ratio is also a database-capacity metric.

# 43. Cache memory estimation

Suppose we cache:

``` text
100 million URLs
```

Average logical payload:

``` text
1 KB
```

Raw payload:

``` text
100M × 1 KB ≈ 100 GB
```

Actual Redis memory is higher because of:

-   Key memory
-   Object/data-structure overhead
-   Expiration metadata
-   Allocator overhead
-   Replicas
-   Operational headroom

Therefore never size Redis from value bytes alone.

# 44. Cache and database protection

The cache exists partly to protect the database.

But if the cache fails, that protection disappears.

Use:

``` text
Timeouts
Bounded concurrency
Circuit breakers
Rate limiting
Backpressure
```

Goal:

``` text
Redis outage
   |
Controlled DB load
   |
Service remains partially available
```

rather than:

``` text
Redis outage
   |
Unbounded DB traffic
   |
DB outage
   |
Whole system outage
```

# 45. Cache failure decision tree

``` text
Request
  |
Redis available?
 / \
YES NO
 |   |
GET  Is DB capacity safe?
 |    /        \
HIT  YES        NO
 |    |          |
302  DB      controlled failure
      |
      v
  populate cache
      |
     302
```

# 46. Observability for cache

Monitor:

``` text
Hit ratio
Miss ratio
GET latency
SET latency
Commands/sec
Memory usage
Evictions
Expired keys
Connections
Errors
Replication lag
Shard distribution
Hot keys
```

Useful alert examples:

``` text
Hit ratio suddenly drops
Memory approaches limit
Evictions spike
Redis latency rises
Replica falls behind
Connection errors increase
Shard becomes unusually hot
```

# 47. Security of Redis

Redis should normally be inside a private network boundary rather than
publicly exposed.

``` text
Internet
   |
Load Balancer
   |
API
   |
Private Redis
```

Use appropriate:

-   Authentication/ACLs
-   TLS where required
-   Network isolation
-   Secret management
-   Least privilege

Do not cache sensitive data unless there is a clear reason and
appropriate controls.

# 48. Complete URL-shortener cache architecture

``` text
                           Users
                             |
                             v
                       Load Balancer
                             |
                   +---------+---------+
                   |         |         |
                  API       API       API
                   |         |         |
                   +---------+---------+
                             |
                             v
                       Redis Cluster
                    /       |       \
                 Shard1   Shard2   Shard3
                             |
                          MISS only
                             |
                             v
                      Read Replicas
                             |
                             v
                         Primary DB
```

## Read path

``` text
GET /aZ91k
    |
    v
   API
    |
 Redis GET
  /      \
HIT      MISS
 |          |
 v          v
302       DB read
             |
             v
          Redis SET
             |
             v
            302
```

## Write path

``` text
POST /api/urls
       |
       v
      API
       |
Generate ID
       |
Primary DB INSERT
       |
Base62 encode ID
       |
Redis SET url:code -> originalUrl
       |
Return short URL
```

# 49. What if Redis is down?

Do not simply say:

> "Read from DB."

At large scale that can overload the DB.

Instead:

``` text
Redis unavailable
      |
      +--> timeout quickly
      +--> circuit breaker
      +--> limit concurrent DB reads
      +--> rate limit where appropriate
      +--> use DB only within safe capacity
      +--> return controlled errors when necessary
```

# 50. What if a hot key expires?

Bad:

``` text
1M requests
    |
Redis MISS
    |
1M DB requests
```

Better:

``` text
1M requests
    |
Redis MISS
    |
Single-flight
    |
ONE DB request
    |
Redis SET
    |
all callers receive result
```

# 51. What if millions of keys expire together?

Use:

``` text
TTL jitter
Staggered TTLs
Refresh-ahead for hot data
Cache redundancy
DB protection
```

# 52. What if the cache is full?

The configured Redis memory/eviction policy determines behavior.

Monitor memory and evictions. If important entries are being evicted too
aggressively, options include:

-   Increase cache capacity
-   Improve key/TTL strategy
-   Remove unnecessary cached data
-   Use appropriate eviction policy
-   Add shards
-   Improve hot-data strategy

# 53. What if the database is already overloaded?

Do not increase retries.

Instead:

``` text
DB overloaded
   |
Circuit breaker / concurrency limit
   |
Reduce load
   |
Allow DB to recover
```

A cache miss should not become an excuse for unlimited database traffic.

# 54. Interview comparison table

  Concept             Meaning
  ------------------- ----------------------------------------------------------
  Cache-aside         Application handles cache read and miss loading
  Read-through        Cache loads backing store on miss
  Write-through       Cache synchronously updates backing store
  Write-behind        Cache writes first, DB later
  Refresh-ahead       Refresh before expiration
  TTL                 Time-based expiration
  LRU                 Evict least recently used
  LFU                 Evict least frequently used
  Local cache         Per-process cache
  Distributed cache   Shared cache
  Stampede            Many requests reload same missing key
  Penetration         Requests for nonexistent data repeatedly hit backend
  Breakdown           Hot key becomes unavailable and causes concentrated load
  Avalanche           Large-scale cache expiry/failure causes backend spike
  Replication         Copies data
  Sharding            Splits data

# 55. CAP and caching

Do not say:

> "Redis is automatically AP and SQL is automatically CP."

CAP is about a distributed system's behavior during a network partition.
The exact consistency/availability behavior depends on the specific
system, configuration and operation.

Replication lag alone is not automatically a CAP partition.

For the URL shortener:

``` text
DB = source of truth
Redis = performance layer
```

We explicitly design what happens when replicas/cache nodes cannot
communicate.

# 56. Recommended caching strategy for our URL shortener

``` text
Cache type:
Distributed cache

Technology:
Redis

Pattern:
Cache-aside

Read:
Redis -> DB on miss

Write:
Primary DB -> Redis populate

Expiration:
TTL

Invalidation:
Explicit delete for mutable data + TTL safety net

Hot keys:
Local cache/CDN/request coalescing when justified

Stampede:
Single-flight/lock + TTL jitter + refresh-ahead

Failure:
Failover/degrade carefully; protect DB

Scaling:
Redis replication + sharding

Monitoring:
Hit ratio + latency + memory + evictions + errors + hot keys
```

# 57. Complete example

Suppose the user creates:

``` text
https://www.example.com/products/123
```

### Create

``` text
POST
 |
API
 |
Snowflake ID = 1234567890123
 |
DB INSERT
 |
Base62(1234567890123)
 |
shortCode = aZ91k
 |
Redis:
url:aZ91k -> https://www.example.com/products/123
 |
response
```

### Redirect

``` text
GET /aZ91k
 |
API
 |
Redis GET
 |
HIT
 |
302 Location: https://www.example.com/products/123
```

### First request after cache loss

``` text
GET /aZ91k
 |
Redis MISS
 |
Read DB
 |
Redis SET
 |
302
```

### Hot key

``` text
Millions of GET /aZ91k
 |
Redis / CDN / L1 cache
 |
avoid repeated DB work
```

# 58. 60-second interview answer

If asked, "How will you use caching in the URL shortener?":

> "The system is read-heavy, so I would use Redis as a distributed cache
> in front of the database. The key would be the short code and the
> value would be the original URL, so a cache hit can immediately return
> the redirect without another DB lookup. I'd use cache-aside for reads:
> check Redis first, on a miss read from the appropriate database path,
> populate Redis with a TTL, and return the redirect. On creation I'd
> durably write the mapping to the primary database and populate Redis
> so the first read is also fast. Because Redis is a cache and not the
> source of truth, a Redis outage must not blindly send all traffic to
> the DB; I'd use timeouts, circuit breakers, bounded concurrency and
> rate limiting to protect the database. For hot keys and stampedes I'd
> consider request coalescing, TTL jitter, refresh-ahead, local caching
> or CDN depending on the traffic pattern. At scale I'd use Redis
> replication and sharding, and monitor hit ratio, latency, memory,
> evictions, errors and hot keys."

# 59. Final mental model

``` text
                           CACHING
                              |
             +----------------+----------------+
             |                |                |
            WHERE            HOW              WHEN
             |                |                |
       Browser/CDN       Cache-aside          TTL
       Local/Redis       Read-through          Delete
       Multi-level       Write-through         Refresh
                          Write-behind
             |                |
             +----------------+
                      |
                 FAILURE MODES
                      |
       +--------------+--------------+
       |              |              |
   Stampede        Hot Key       Avalanche
       |              |              |
  Coalesce       L1/CDN        TTL jitter
       |              |              |
       +--------------+--------------+
                      |
                  SCALING
                      |
             +--------+--------+
             |                 |
        Replication         Sharding
```

The most important path:

``` text
                    GET /aZ91k
                         |
                         v
                        API
                         |
                 Redis: url:aZ91k
                    /          \
                  HIT          MISS
                   |             |
                   v             v
              originalUrl        DB
                   |             |
                   |             v
                   |         originalUrl
                   |             |
                   |             v
                   |         Redis SET
                   |             |
                   +------->-----+
                         |
                         v
                       302
```

> **The cache is a performance optimization; the database is the source
> of truth.**
