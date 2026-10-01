# Bloomberg URL Shortener — Unique ID Generation Deep Dive

## 1. The real problem
The pipeline is:
```text
Long URL -> Unique ID -> Base62 -> Short Code
```
Base62 is only representation. The hard question is how to create the unique ID safely under concurrency.

## 2. SQL Identity
Simple design:
```text
API -> SQL Server -> generated ID
```
Example:
```text
125 -> google.com
126 -> amazon.com
```
EF Core:
```csharp
var mapping = new UrlMapping { OriginalUrl = url };
db.UrlMappings.Add(mapping);
await db.SaveChangesAsync();
long id = mapping.Id;
```
Multiple API servers can insert concurrently because the database coordinates ID generation.

## 3. Gaps are okay
Identity/sequence values can have gaps after rollback/crash:
```text
125, 126, 127, 129
```
For a URL shortener we need uniqueness, not gapless numbering.

## 4. SQL Sequence
A sequence is a standalone DB number generator:
```sql
SELECT NEXT VALUE FOR UrlSequence;
```
Identity is tied to insertion; sequence can be explicitly requested.

## 5. Why DB generation can become a bottleneck
At high write volume:
```text
API1 \
API2  \
API3   ---> one primary DB
...   /
API100/
```
Every creation depends on the DB for ID generation and durable insertion.

## 6. GUID
Each API can generate locally:
```text
API1 -> GUID A
API2 -> GUID B
API3 -> GUID C
```
Example:
```text
550e8400-e29b-41d4-a716-446655440000
```
Advantages:
- decentralized
- no central ID service
- extremely low collision probability

Disadvantages:
- long representation
- less naturally ordered
- can be less friendly for indexed/clustered storage
- not ideal for compact public URLs

## 7. Snowflake-style IDs
A distributed ID contains:
```text
+----------------+-------------+------------+
|   Timestamp    | Worker ID   |  Sequence  |
+----------------+-------------+------------+
```
It answers:
```text
WHEN?  WHO?  WHICH REQUEST?
```

## 8. Common 64-bit layout
A widely known layout is:
```text
1 sign | 41 timestamp | 10 worker | 12 sequence
```
Total:
```text
64 bits
```

Diagram:
```text
63                                      0
+---+--------------------------------+----------+------------+
| 0 |          Timestamp             |  Worker  |  Sequence  |
+---+--------------------------------+----------+------------+
    41 bits                         10 bits     12 bits
```

## 9. Timestamp
Usually:
```text
timestamp = current UTC milliseconds - custom epoch
```
41 bits provide about 69 years of millisecond values. The actual lifetime depends on the chosen epoch.

## 10. Worker ID
10 bits:
```text
2^10 = 1024
```
possible worker IDs under this allocation.

The deployment must ensure two active generators do not share a worker ID.

## 11. Sequence
12 bits:
```text
2^12 = 4096
```
IDs per worker per millisecond under this allocation.

Same millisecond:
```text
Worker 1, sequence 0
Worker 1, sequence 1
Worker 1, sequence 2
```

## 12. Why uniqueness works
Different workers:
```text
timestamp=5000, worker=1, seq=0
timestamp=5000, worker=2, seq=0
```
Different IDs.

Same worker:
```text
timestamp=5000, worker=1, seq=0
timestamp=5000, worker=1, seq=1
```
Different IDs.

## 13. Packing the bits
With 10 worker bits and 12 sequence bits:
```text
ID =
(timestamp << 22)
|
(workerId << 12)
|
sequence
```
because:
```text
10 + 12 = 22
```

## 14. C# conceptual generator
```csharp
public class SnowflakeIdGenerator
{
    private const int WorkerBits = 10;
    private const int SequenceBits = 12;
    private const long MaxWorkerId = (1L << WorkerBits) - 1;
    private const long SequenceMask = (1L << SequenceBits) - 1;

    private readonly long _workerId;
    private long _sequence;
    private long _lastTimestamp = -1;

    private readonly DateTime _epoch =
        new(2020, 1, 1, 0, 0, 0, DateTimeKind.Utc);

    public SnowflakeIdGenerator(long workerId)
    {
        if (workerId < 0 || workerId > MaxWorkerId)
            throw new ArgumentOutOfRangeException(nameof(workerId));

        _workerId = workerId;
    }

    public lock object SyncRoot { get; } = new();

    public long NextId()
    {
        lock (SyncRoot)
        {
            long timestamp = CurrentTimestamp();

            if (timestamp < _lastTimestamp)
                throw new InvalidOperationException("Clock moved backwards.");

            if (timestamp == _lastTimestamp)
            {
                _sequence = (_sequence + 1) & SequenceMask;

                if (_sequence == 0)
                    timestamp = WaitForNextMillisecond(_lastTimestamp);
            }
            else
            {
                _sequence = 0;
            }

            _lastTimestamp = timestamp;

            return (timestamp << 22)
                 | (_workerId << 12)
                 | _sequence;
        }
    }

    private long CurrentTimestamp() =>
        (long)(DateTime.UtcNow - _epoch).TotalMilliseconds;

    private long WaitForNextMillisecond(long lastTimestamp)
    {
        long timestamp;
        do
        {
            timestamp = CurrentTimestamp();
        }
        while (timestamp <= lastTimestamp);

        return timestamp;
    }
}
```

This is a learning implementation; production systems need careful worker-ID allocation and clock handling.

## 15. Sequence overflow
If sequence reaches 4095 in one millisecond:
```text
wait -> next millisecond -> sequence 0
```

## 16. Clock rollback
If:
```text
last timestamp = 5000
current timestamp = 4998
```
timestamp moved backwards.

Possible policies:
- wait
- fail temporarily
- logical clock
- disciplined time synchronization

## 17. Worker ID assignment
Worker IDs can come from:
- static configuration
- container/task identity
- a coordination/allocation service
- platform identity

The important invariant is:
```text
two active generators must not use the same worker ID
```

## 18. Snowflake + Base62
```text
Long URL
   |
   v
Snowflake generator
   |
   v
123456789012345
   |
   v
Base62.Encode()
   |
   v
aZ91k
   |
   v
https://short.ly/aZ91k
```

Snowflake solves distributed uniqueness.
Base62 solves compact representation.

## 19. Comparison
| Mechanism | Central DB per ID? | Distributed | Compact | Complexity |
|---|---:|---:|---:|---:|
| Identity | Yes | No | Yes | Low |
| Sequence | Yes | Limited | Yes | Low/Medium |
| GUID | No | Yes | Less ideal | Low |
| Snowflake-style | No | Yes | Yes | Medium/High |

## 20. Interview answer
> For a simple implementation I can use a SQL identity or sequence. If ID generation becomes a bottleneck, I can use a Snowflake-style distributed ID generator combining timestamp, worker ID and sequence. That lets API instances generate IDs independently, provided worker IDs are unique and clock behavior is controlled. I then Base62-encode the numeric ID for the public short code.
