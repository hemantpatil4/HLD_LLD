# Bloomberg URL Shortener — Redis & Caching Deep Dive

## 1. Why caching?
A URL shortener is typically read-heavy: many users can click the same short URL repeatedly.

Without cache:
```text
User -> Load Balancer -> API -> Database -> Original URL -> 302
```

With Redis:
```text
User -> Load Balancer -> API -> Redis
                                | HIT  -> 302
                                | MISS -> DB -> Redis SET -> 302
```

**Core principle:** the database is the source of truth; Redis is a performance layer.

## 2. Cache types
### Client cache
Browser/device-local storage.

### CDN cache
Edge cache close to users:
```text
User -> Edge -> Origin
```

### In-memory application cache
.NET `IMemoryCache`.
```text
API1 -> Cache A
API2 -> Cache B
```
Very fast, but not shared.

### Distributed cache
Redis:
```text
API1 \
API2  ---> Redis
API3 /
```

## 3. Cache-aside
The application owns the lookup:
```text
Request
  |
  v
Redis GET
 /      \
HIT     MISS
 |        |
 v        v
Return   DB
          |
          v
       Redis SET
          |
          v
        Return
```

Basic C#:
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

## 4. Cache key design
Prefer namespaced keys:
```text
url:21 -> https://google.com
```
Do not cache only:
```text
21 -> 125
```
because that still requires a DB lookup. Cache the final value needed by the redirect.

## 5. TTL
TTL = Time To Live.
```text
url:21 -> URL
TTL = 1 hour
```
Expiration removes the cache copy, not the DB row.

## 6. Invalidation
If a URL can change:
```text
Update DB
   |
   v
Delete Redis key
   |
   v
Next request -> MISS -> DB -> SET
```
C#:
```csharp
await db.SaveChangesAsync();
await redis.KeyDeleteAsync($"url:{code}");
```

For immutable mappings, invalidation is much simpler.

## 7. Other cache patterns
### Write-through
```text
App -> Cache -> DB
```
### Write-back/write-behind
```text
App -> Cache -> later DB
```
Faster, but durability is harder.
### Read-through
```text
App -> Cache -> DB on miss
```
The cache abstraction owns loading.
### Cache-aside
Application explicitly controls the miss path; this is the main pattern for our design.

## 8. Eviction
Common policies:
- LRU — least recently used
- LFU — least frequently used
- FIFO — first in, first out
- TTL expiration

## 9. Cache stampede
If a hot key expires:
```text
100,000 requests
      |
      v
  MISS MISS MISS
      |
      v
100,000 DB requests
```
Mitigations:
- request coalescing/single-flight
- distributed lock
- TTL jitter
- prewarming
- stale-while-revalidate

Concept:
```text
Many requests -> one DB load -> Redis populated -> others reuse
```

## 10. Hot keys
One URL may receive huge traffic:
```text
url:21
```
Mitigations:
- Redis replicas
- local in-memory cache for ultra-hot values
- edge/CDN caching where appropriate
- traffic shaping

## 11. Redis replication vs sharding
Replication:
```text
Primary
 /    \
R1    R2
```
Copies the dataset for redundancy/failover.

Sharding:
```text
Redis Cluster
 /   |   \
S1  S2   S3
```
Splits the dataset for scale.

Combined:
```text
Shard1: P/R
Shard2: P/R
Shard3: P/R
```

## 12. Redis failure
Naive fallback:
```text
Redis DOWN -> DB
```
At 1M redirects/sec this can overwhelm the DB.

Use:
- Redis redundancy/failover
- short timeouts
- circuit breaker
- bounded DB concurrency
- rate limiting
- controlled degradation

## 13. .NET abstraction
A service abstraction keeps Redis details out of business logic:
```csharp
public interface IUrlCache
{
    Task<string?> GetAsync(string code);
    Task SetAsync(string code, string url, TimeSpan ttl);
    Task RemoveAsync(string code);
}
```

The implementation can use `IDistributedCache` or a Redis client such as StackExchange.Redis.

## 14. Metrics
Track:
```text
Hit ratio = hits / total cache requests
Miss ratio
Redis latency
Redis errors
Memory usage
Evictions
Hot keys
Connections
Replication lag
```

## 15. Interview answer
> I would use Redis as a distributed cache because redirects are read-heavy. The short code is the cache key and the original URL is the value. I would use cache-aside: check Redis first, read the database on a miss, populate Redis, and redirect. TTL handles expiry, and if mappings can change I invalidate the cache after DB updates. At scale I would use Redis replication/sharding and protect the database from cache outages and stampedes.
