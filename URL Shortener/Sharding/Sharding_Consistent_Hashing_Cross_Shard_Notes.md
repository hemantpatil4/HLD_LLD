# Sharding --- Consistent Hashing, Node Arrival, Virtual Nodes & Cross-Shard Queries

## Interview Preparation Notes

These notes explain the concepts from first principles using one
continuous example.

------------------------------------------------------------------------

# 1. Background: Why Do We Need Sharding?

Suppose an application has a very large `Customer` table:

``` text
CustomerId -> Customer data
```

Initially, everything may live on one database:

``` text
Application
     |
     v
    DB
```

As the system grows, one database may become a bottleneck because of:

-   CPU
-   Memory
-   Disk I/O
-   Storage
-   Network
-   Number of connections
-   Read/write throughput

We can scale vertically by making the database bigger, but eventually we
need horizontal scaling.

## Horizontal Scaling

Instead of one large database:

``` text
              Application
                   |
                   v
                  DB
```

we distribute the data:

``` text
                  Application
                       |
                  Shard Router
                       |
          +------------+------------+
          |            |            |
         DB1          DB2          DB3
```

This is called **sharding**.

------------------------------------------------------------------------

# 2. What Is Sharding?

Sharding means splitting a large logical dataset across multiple
physical database instances.

For example:

``` text
DB1 -> some customers
DB2 -> some customers
DB3 -> some customers
```

The application still thinks it has one logical `Customer` dataset, but
physically the records are distributed.

The most important question becomes:

> Given a record key, how do I determine which shard owns it?

This is the job of the **shard-routing mechanism**.

------------------------------------------------------------------------

# 3. Shard Key

The value used to determine the shard is called the **shard key** or
**partition key**.

For example:

``` text
customerId
```

We might use:

``` text
customerId -> hashing/routing -> shard
```

A good shard key should generally:

-   distribute data reasonably evenly
-   distribute traffic reasonably evenly
-   appear frequently in important queries
-   avoid creating a single hot partition
-   allow targeted routing when possible

Shard-key selection is one of the most important decisions in a sharded
architecture.

------------------------------------------------------------------------

# 4. Naive Approach: `hash(key) % N`

Suppose we have three databases:

``` text
DB1
DB2
DB3
```

A simple strategy is:

``` text
shard = hash(key) % 3
```

Example:

``` text
User 101
hash(101) = 10

10 % 3 = 1

=> DB1
```

Another:

``` text
User 102
hash(102) = 17

17 % 3 = 2

=> DB2
```

Another:

``` text
User 103
hash(103) = 22

22 % 3 = 1

=> DB1
```

This looks simple.

But there is a major problem when the number of shards changes.

------------------------------------------------------------------------

# 5. Problem With `hash(key) % N`

Initially:

``` text
N = 3
```

For:

``` text
hash(User101) = 10
```

we get:

``` text
10 % 3 = 1
=> DB1
```

Now suppose we add DB4.

We now have:

``` text
N = 4
```

The same key becomes:

``` text
10 % 4 = 2
=> DB2
```

The record moved:

``` text
DB1 -> DB2
```

Another example:

``` text
hash(User102) = 17

17 % 3 = 2
=> DB2

17 % 4 = 1
=> DB1
```

Again, it moved.

So adding one database can cause a very large percentage of keys to map
to different shards.

That means potentially huge data movement.

------------------------------------------------------------------------

# 6. Why Large-Scale Data Movement Is Bad

Imagine:

``` text
100 TB data
10 database shards
```

Now add an 11th shard.

With modulo hashing, many records may need to be redistributed.

That causes:

-   network traffic
-   disk I/O
-   CPU usage
-   replication pressure
-   cache invalidation
-   longer migrations
-   increased latency
-   possible production instability

We want a routing strategy where:

> Adding/removing a node causes as little remapping as possible.

This is where **consistent hashing** comes in.

------------------------------------------------------------------------

# 7. Consistent Hashing

Instead of:

``` text
hash(key) % numberOfNodes
```

we create a logical **hash ring**.

Imagine the hash space is:

``` text
0 -> 1 -> 2 -> ... -> 99 -> back to 0
```

Because it wraps around, it forms a circle.

Conceptually:

``` text
                 0
                 |
          90 ----+---- 10
                 |
          80     |     20
                 |
          70 ----+---- 30
                 |
          60 ----+---- 40
                 |
                50
```

The actual system may use a much larger hash space, such as 32-bit or
64-bit values. The small `0..99` range is just for understanding.

------------------------------------------------------------------------

# 8. Put Database Nodes on the Ring

Suppose:

``` text
DB1 -> position 20
DB2 -> position 50
DB3 -> position 80
```

Ring:

``` text
                 0
                 |
             DB1 (20)
                 |
                 |
             DB2 (50)
                 |
                 |
             DB3 (80)
                 |
                 |
                 +---- back to 0
```

The important rule is:

> A key belongs to the first database node encountered when moving
> clockwise from the key's hash position.

------------------------------------------------------------------------

# 9. Actual Key Example

Suppose:

``` text
hash(User101) = 25
```

The next database clockwise is:

``` text
DB2 at 50
```

Therefore:

``` text
User101 -> DB2
```

Another:

``` text
hash(User102) = 65
```

Next database clockwise:

``` text
DB3 at 80
```

Therefore:

``` text
User102 -> DB3
```

Another:

``` text
hash(User103) = 85
```

There is no node after 85.

The ring wraps around:

``` text
85 -> ... -> 99 -> 0 -> ... -> 20
```

So:

``` text
User103 -> DB1
```

------------------------------------------------------------------------

# 10. Ownership Ranges

With:

``` text
DB1 = 20
DB2 = 50
DB3 = 80
```

the ownership is:

``` text
20 -> 50  = DB2
50 -> 80  = DB3
80 -> 20  = DB1
```

This is the key mental model.

A database owns a range of hash positions.

------------------------------------------------------------------------

# 11. Adding a New Node

Now add:

``` text
DB4
```

Suppose:

``` text
hash("DB4") = 60
```

So:

``` text
DB4 -> 60
```

The ring becomes:

``` text
DB1 = 20
DB2 = 50
DB4 = 60
DB3 = 80
```

Previously:

``` text
50 -> 80 = DB3
```

Now:

``` text
50 -> 60 = DB4
60 -> 80 = DB3
```

Therefore DB4 only takes ownership of:

``` text
50 -> 60
```

from DB3.

------------------------------------------------------------------------

# 12. Actual Data Movement

Suppose DB3 currently has:

``` text
User A -> hash 52
User B -> hash 55
User C -> hash 58
User D -> hash 65
User E -> hash 75
```

Before DB4:

``` text
52 -> DB3
55 -> DB3
58 -> DB3
65 -> DB3
75 -> DB3
```

After DB4 joins:

``` text
52 -> DB4
55 -> DB4
58 -> DB4
65 -> DB3
75 -> DB3
```

So only:

``` text
User A
User B
User C
```

need to move.

The rest stay on DB3.

This is the main advantage of consistent hashing.

------------------------------------------------------------------------

# 13. Consistent Hashing Does Not Mean No Data Movement

This is an important interview distinction.

Do NOT say:

> "Consistent hashing means data never moves."

Instead say:

> "Consistent hashing minimizes the amount of data that needs to be
> remapped when the cluster membership changes."

Some data must move when a new node takes ownership of a range.

------------------------------------------------------------------------

# 14. How Does the Router Work?

Conceptually:

``` text
Request
   |
   v
Partition Key
   |
   v
hash(key)
   |
   v
position on ring
   |
   v
first node clockwise
   |
   v
Shard
```

Example:

``` text
customerId = 123
        |
        v
hash(123) = 55
        |
        v
first node clockwise
        |
        v
DB4
```

------------------------------------------------------------------------

# 15. Implementation Idea

Conceptually, the ring can be represented as:

``` csharp
SortedDictionary<long, Server>
```

For example:

``` text
20 -> DB1
50 -> DB2
60 -> DB4
80 -> DB3
```

For a key:

``` csharp
long hash = Hash(key);
```

Find the first ring position greater than or equal to the hash.

If none exists, wrap around to the first position.

Conceptually:

``` csharp
Server GetServer(string key)
{
    long hash = Hash(key);

    // Find first node >= hash.
    // If none exists, wrap around to first node.

    return selectedServer;
}
```

A real implementation should use an efficient ordered structure and
binary search rather than scanning every node.

Typical lookup complexity is approximately:

``` text
O(log N)
```

where `N` is the number of ring positions.

------------------------------------------------------------------------

# 16. New Node Arrival in Production

Consistent hashing tells us the new ownership.

But actual data still needs to be physically migrated.

A production flow can look like:

``` text
1. Add DB4
       |
2. Hash DB4 identity
       |
3. Place DB4 on the ring
       |
4. Determine the ranges DB4 owns
       |
5. Identify records in those ranges
       |
6. Copy records to DB4
       |
7. Synchronize writes during migration
       |
8. Verify DB4
       |
9. Switch routing
       |
10. Remove old copies when safe
```

------------------------------------------------------------------------

# 17. Migration Problem

Suppose DB4 should own:

``` text
50 -> 60
```

but migration is still running.

The router may already know:

``` text
hash(User101) = 55
```

and therefore:

``` text
User101 -> DB4
```

But User101 may still physically exist only on DB3.

So:

``` text
Router -> DB4
DB4 -> record not found
DB3 -> record exists
```

This is why real systems need a migration strategy.

Possible mechanisms include:

-   temporary dual reads
-   dual writes
-   change-data capture
-   replication
-   write-ahead logs
-   migration checkpoints
-   version numbers
-   idempotent retries

The exact mechanism depends on the system's consistency and availability
requirements.

------------------------------------------------------------------------

# 18. Why Writes During Migration Are Difficult

Suppose User101 is being migrated:

``` text
DB3 -> DB4
```

At the same time:

``` text
User101 changes email
```

If the write goes only to DB3:

``` text
DB3 = new value
DB4 = old value
```

After migration, DB4 could contain stale data.

One conceptual approach is dual-write:

``` text
Application
    |
    +----> DB3
    |
    +----> DB4
```

But dual-write itself introduces failure cases:

``` text
DB3 write succeeds
DB4 write fails
```

Now the two copies differ.

More sophisticated systems can use CDC, logs, retries, versioning, or
replication to make migration safe.

------------------------------------------------------------------------

# 19. Safe Migration Principle

A useful mental model is:

``` text
DB3 = source
DB4 = destination
```

Do not remove the source copy immediately.

Only fully switch ownership after:

``` text
data copied
+
changes synchronized
+
verification completed
+
routing updated
```

This protects against:

-   migration failure
-   destination failure
-   partial transfer
-   data loss

------------------------------------------------------------------------

# 20. What If the New Node Fails?

Suppose:

``` text
DB4
```

fails during migration.

If DB3 still has the source data:

``` text
DB3
 |
 +---- source data still safe
 |
 +----> DB4
```

we can retry/resume migration later.

The important principle is:

> Do not delete the source data until the new owner is confirmed safe
> and authoritative.

------------------------------------------------------------------------

# 21. Virtual Nodes

Consistent hashing solves one problem:

> Minimize movement when nodes change.

But there is another problem:

> How do we make sure the ring is balanced?

If we put only one position per physical server, hashing may produce an
unlucky distribution.

For example:

``` text
DB1 -> 10
DB2 -> 11
DB3 -> 90
```

Ownership could become roughly:

``` text
DB1 -> 20%
DB2 -> 1%
DB3 -> 79%
```

That is a bad distribution.

------------------------------------------------------------------------

# 22. Virtual Nodes Solve the Distribution Problem

Instead of placing each physical server once:

``` text
DB1 -> one position
DB2 -> one position
DB3 -> one position
```

we place each physical server at many positions.

For example:

``` text
DB1:
10
40
75

DB2:
20
50
90

DB3:
30
60
80
```

The ring becomes:

``` text
10 -> DB1
20 -> DB2
30 -> DB3
40 -> DB1
50 -> DB2
60 -> DB3
75 -> DB1
80 -> DB3
90 -> DB2
```

These are **virtual nodes**.

------------------------------------------------------------------------

# 23. What Is a Virtual Node?

A virtual node is:

> A logical position on the hash ring that maps to a physical server.

For example:

``` text
DB1
 |
 +-- Virtual Node 1 -> position 10
 +-- Virtual Node 2 -> position 40
 +-- Virtual Node 3 -> position 75
```

There are still only three physical servers.

``` text
Physical servers:
DB1
DB2
DB3
```

The virtual nodes are only logical ring positions.

------------------------------------------------------------------------

# 24. Actual Request With Virtual Nodes

Suppose:

``` text
hash(User123) = 43
```

Ring:

``` text
40 -> DB1
50 -> DB2
60 -> DB3
```

Move clockwise from 43:

``` text
43 -> 50
```

Position 50 belongs to a virtual node of DB2.

Therefore:

``` text
User123 -> DB2
```

The final destination is still the physical server.

``` text
User123
   |
hash = 43
   |
VNode at 50
   |
DB2
```

------------------------------------------------------------------------

# 25. Why Virtual Nodes Improve Distribution

With one position:

``` text
DB1 -> one large range
DB2 -> one large range
DB3 -> one large range
```

A bad placement can create very uneven ownership.

With many positions:

``` text
DB1 -> many small ranges
DB2 -> many small ranges
DB3 -> many small ranges
```

The ranges are spread around the ring.

Statistically, the ownership becomes much more balanced.

For example:

``` text
DB1 -> ~33%
DB2 -> ~34%
DB3 -> ~33%
```

Exact values depend on the number and placement of virtual nodes.

------------------------------------------------------------------------

# 26. Virtual Nodes Also Help During Node Failure

Suppose DB2 has many virtual nodes:

``` text
DB2-V1
DB2-V2
DB2-V3
...
```

Now DB2 fails.

Its virtual nodes disappear.

The previously owned ranges are distributed across multiple surviving
physical servers.

Instead of:

``` text
DB2 fails
   |
one neighboring server
   |
huge range inherited
```

we get:

``` text
DB2 fails
   |
many small ranges become available
   |
multiple servers inherit those ranges
```

This reduces the chance of concentrating all the recovery load on one
server.

------------------------------------------------------------------------

# 27. Virtual Nodes and Different Server Capacities

Virtual nodes can also represent different capacities.

Suppose:

``` text
DB1 = powerful
DB2 = powerful
DB3 = smaller
```

We may use:

``` text
DB1 -> 100 virtual nodes
DB2 -> 100 virtual nodes
DB3 -> 50 virtual nodes
```

Conceptually:

``` text
DB1 -> ~40%
DB2 -> ~40%
DB3 -> ~20%
```

This is commonly described as **weighted consistent hashing**.

------------------------------------------------------------------------

# 28. Adding a Node With Virtual Nodes

Suppose DB4 joins.

Give it multiple virtual nodes:

``` text
DB4:
35
65
95
```

Now DB4 doesn't necessarily take one large continuous region.

It takes multiple smaller ranges from different parts of the ring.

This makes rebalancing more distributed.

Important:

> Virtual nodes do not eliminate data movement.

They make ownership and rebalancing more balanced.

------------------------------------------------------------------------

# 29. Consistent Hashing vs Virtual Nodes

Keep these concepts separate.

### Consistent Hashing

Problem solved:

``` text
How do I minimize remapping when nodes change?
```

Mechanism:

``` text
Hash ring
+
clockwise ownership
```

------------------------------------------------------------------------

### Virtual Nodes

Problem solved:

``` text
How do I make ring ownership more evenly distributed?
```

Mechanism:

``` text
Multiple logical positions per physical node
```

------------------------------------------------------------------------

### Rebalancing

Problem solved:

``` text
How do I physically move data after ownership changes?
```

Mechanism:

``` text
Data migration / replication / CDC / etc.
```

------------------------------------------------------------------------

# 30. Cross-Shard Queries

Now suppose we have:

``` text
DB1
DB2
DB3
DB4
```

and customers are distributed across them.

A request such as:

``` sql
SELECT *
FROM Customer
WHERE customerId = 101;
```

is easy if:

``` text
customerId
```

is the shard key.

The router calculates:

``` text
hash(101)
   |
   v
DB3
```

Only DB3 is queried.

This is a **targeted query**.

------------------------------------------------------------------------

# 31. Cross-Shard Query Example

Now consider:

``` sql
SELECT *
FROM Customer
WHERE name = 'Hemant';
```

Suppose the shard key is `customerId`.

The name does not tell us which shard contains the record.

So we may need:

``` text
              Query
                |
             Router
                |
       +--------+--------+--------+
       |        |        |        |
      DB1      DB2      DB3      DB4
```

This is a **scatter-gather query**.

------------------------------------------------------------------------

# 32. Scatter-Gather

### Scatter

Send the query to multiple shards:

``` text
Query
 |
 +--> DB1
 +--> DB2
 +--> DB3
 +--> DB4
```

### Gather

Collect the responses:

``` text
DB1 --\
DB2 ---\
DB3 ----> Router -> final result
DB4 ---/
```

Therefore:

> Scatter = distribute the query.

> Gather = combine the results.

------------------------------------------------------------------------

# 33. Why Cross-Shard Queries Are Expensive

Suppose each database normally responds in around 50 ms.

If we query 10 shards in parallel, latency is approximately determined
by the slowest shard:

``` text
DB1 = 40ms
DB2 = 50ms
DB3 = 45ms
DB4 = 70ms
...
DB10 = 80ms

Overall ≈ 80ms
```

It is not necessarily:

``` text
10 × 50 = 500ms
```

because requests can execute in parallel.

But we still pay for:

-   multiple network requests
-   multiple DB operations
-   more connections
-   more CPU
-   result merging
-   higher probability of one slow/failing shard

------------------------------------------------------------------------

# 34. Cross-Shard Aggregation

Consider:

``` sql
SELECT COUNT(*)
FROM Customers;
```

Suppose:

``` text
DB1 -> 2.5M
DB2 -> 2.7M
DB3 -> 2.4M
DB4 -> 2.4M
```

The router combines:

``` text
2.5M
+ 2.7M
+ 2.4M
+ 2.4M
= 10M
```

The general pattern is:

``` text
Local aggregation
      |
      v
Global aggregation
```

------------------------------------------------------------------------

# 35. SUM

For:

``` sql
SELECT SUM(amount)
FROM Orders;
```

Each shard returns:

``` text
DB1 -> 1000
DB2 -> 2000
DB3 -> 1500
DB4 -> 2500
```

Global result:

``` text
1000 + 2000 + 1500 + 2500
= 7000
```

------------------------------------------------------------------------

# 36. AVG

AVG needs special care.

Do not simply average shard averages.

Example:

``` text
DB1:
sum = 100
count = 10
avg = 10

DB2:
sum = 100
count = 100
avg = 1
```

Incorrect:

``` text
(10 + 1) / 2 = 5.5
```

Correct:

``` text
total sum   = 100 + 100 = 200
total count = 10 + 100 = 110

global average = 200 / 110
               ≈ 1.82
```

Therefore distributed AVG often requires:

``` text
SUM + COUNT
```

from every shard.

------------------------------------------------------------------------

# 37. Cross-Shard Sorting and Pagination

Consider:

``` sql
SELECT *
FROM Customers
ORDER BY CreatedAt DESC
LIMIT 10;
```

With one DB, the database can sort and return the top 10.

With multiple shards:

``` text
DB1 -> local top 10
DB2 -> local top 10
DB3 -> local top 10
DB4 -> local top 10
```

The router then merges the results.

Example:

``` text
DB1 -> 100, 90, 80
DB2 -> 99, 98, 70
DB3 -> 97, 96, 95
DB4 -> 94, 93, 92
```

Global result begins:

``` text
100
99
98
97
96
95
94
93
92
90
```

The router needs to perform a distributed merge.

This is more complex than a single-database query.

------------------------------------------------------------------------

# 38. Cross-Shard JOIN

Suppose:

``` text
Customers
Orders
```

and we run:

``` sql
SELECT *
FROM Customers c
JOIN Orders o
    ON c.Id = o.CustomerId;
```

With one database, this is straightforward.

With sharding, the data might be:

``` text
Customer 101 -> DB1
Order 5001   -> DB3
```

Now the join crosses databases.

This creates additional network and coordination costs.

------------------------------------------------------------------------

# 39. Co-location

One of the best ways to avoid cross-shard JOINs is to colocate related
data.

Suppose both:

``` text
Customers
Orders
```

are sharded using:

``` text
customerId
```

Then:

``` text
Customer 101 -> DB1

Orders for Customer 101 -> DB1
```

Now:

``` text
DB1
 |
 +-- Customer 101
 |
 +-- Order 1
 +-- Order 2
 +-- Order 3
```

The join can happen locally.

This is called **co-location**.

------------------------------------------------------------------------

# 40. Why Shard-Key Design Matters

Suppose the dominant operation is:

``` text
Get all orders for customer
```

Then:

``` text
customerId
```

is a strong shard-key candidate.

Why?

``` text
customerId
    |
    v
same shard
    |
    +-- Customer
    +-- Orders
```

This allows targeted queries and local joins.

Poor shard-key selection can turn ordinary queries into expensive
distributed queries.

------------------------------------------------------------------------

# 41. Cross-Shard Transactions

Now consider:

``` text
Account A -> DB1
Account B -> DB2
```

We want to transfer ₹1000:

``` text
DB1:
A = A - 1000

DB2:
B = B + 1000
```

What happens if:

``` text
DB1 succeeds
DB2 fails
```

We get:

``` text
A lost ₹1000
B did not receive ₹1000
```

This is a distributed transaction problem.

------------------------------------------------------------------------

# 42. Why Cross-Shard Transactions Are Expensive

A mechanism such as **Two-Phase Commit (2PC)** can coordinate
participants.

Conceptually:

``` text
             Coordinator
              /       \
            DB1       DB2
```

Phase 1:

``` text
"Can you commit?"
```

DB1:

``` text
YES
```

DB2:

``` text
YES
```

Phase 2:

``` text
"Commit."
```

DB1:

``` text
COMMIT
```

DB2:

``` text
COMMIT
```

But this introduces:

-   coordination
-   latency
-   failure handling
-   locking/resource retention
-   availability concerns

Therefore:

> Good shard-key design tries to keep important transactions within one
> shard.

------------------------------------------------------------------------

# 43. Alternatives to Cross-Shard Transactions

Depending on the business requirements, systems may use:

-   Saga patterns
-   compensating actions
-   event-driven workflows
-   asynchronous processing
-   idempotent operations

For example:

``` text
Debit Account A
       |
       v
Publish event
       |
       v
Credit Account B
```

If the second operation fails, a compensating workflow can be triggered.

The exact choice depends on consistency and business requirements.

------------------------------------------------------------------------

# 44. Four Levels of Query Design

A useful interview mental model:

## Level 1 --- Targeted Query

``` text
Query contains shard key
        |
        v
One shard
```

Example:

``` text
customerId = 123
```

Best case.

------------------------------------------------------------------------

## Level 2 --- Co-located Data

``` text
Related entities
      |
      v
Same shard
      |
      v
Local JOIN
```

Example:

``` text
Customer + Orders
```

both shard by `customerId`.

------------------------------------------------------------------------

## Level 3 --- Scatter-Gather

``` text
Query
  |
  +--> DB1
  +--> DB2
  +--> DB3
  +--> DB4
  |
  v
Merge
```

Use when the query cannot be targeted.

------------------------------------------------------------------------

## Level 4 --- Cross-Shard Transaction

``` text
Transaction
     |
     +--> DB1
     |
     +--> DB2
```

Requires distributed coordination.

Try to avoid this where possible.

------------------------------------------------------------------------

# 45. Fan-Out Problem

Suppose the application receives:

``` text
1000 requests/sec
```

and each request must query:

``` text
10 shards
```

Then the database layer receives approximately:

``` text
1000 × 10
=
10,000 shard requests/sec
```

The application sees:

``` text
1000 requests/sec
```

but the backend experiences:

``` text
10,000 operations/sec
```

This is the **fan-out problem**.

Cross-shard fan-out can destroy scalability if used excessively.

Therefore:

> Minimize the number of shards each request touches.

------------------------------------------------------------------------

# 46. Handling Large Fan-Out

Possible techniques include:

-   targeted routing
-   denormalization
-   secondary indexes
-   search systems
-   caching
-   bounded concurrency
-   timeouts
-   circuit breakers
-   backpressure
-   pre-aggregation

For example, a search index can map:

``` text
"Hemant"
    |
    v
CustomerId = 123
    |
    v
Shard Router
    |
    v
DB3
```

Instead of scanning every shard.

------------------------------------------------------------------------

# 47. Cross-Shard Query Architecture

A realistic architecture can look like:

``` text
                         Client
                           |
                           v
                       API Service
                           |
                           v
                      Query Router
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
            DB1           DB2           DB3 ... DBN
```

For targeted queries:

``` text
customerId
    |
    v
Shard Router
    |
    v
ONE shard
```

For unavoidable distributed queries:

``` text
Query
  |
  v
Parallel fan-out
  |
  +---- DB1
  +---- DB2
  +---- DB3
  +---- DBN
  |
  v
Merge / aggregate
  |
  v
Response
```

------------------------------------------------------------------------

# 48. Failure and Timeout Considerations

Suppose:

``` text
DB1 -> 20ms
DB2 -> 30ms
DB3 -> 25ms
DB4 -> 2 seconds
```

A cross-shard request can become slow because of DB4.

Therefore distributed queries often need:

``` text
timeouts
retries carefully
circuit breakers
bounded concurrency
backpressure
```

Whether partial results are acceptable depends on the API.

Some systems can return:

``` text
partial results
```

while others must fail the entire operation.

------------------------------------------------------------------------

# 49. Key Concepts Summary

  Concept                   Main Problem Solved
  ------------------------- --------------------------------------------------
  Sharding                  Scale data/storage/throughput horizontally
  Shard key                 Determine ownership and enable targeted routing
  Consistent hashing        Minimize remapping when nodes change
  Virtual nodes             Improve distribution and rebalance load
  Rebalancing               Physically migrate data after ownership changes
  Targeted query            Query only the responsible shard
  Scatter-gather            Execute unavoidable multi-shard queries
  Co-location               Keep related data on the same shard
  Denormalization           Reduce cross-shard joins
  Cross-shard transaction   Coordinate operations across shards
  Fan-out                   The amplification caused by touching many shards

------------------------------------------------------------------------

# 50. Interview Cheat Sheet

## Consistent Hashing

**Question:** Why not `hash(key) % N`?

**Answer:**

Changing `N` can remap a large percentage of keys.

Consistent hashing places keys and nodes on a logical ring and assigns
each key to the first node clockwise, so node changes affect only
limited ranges.

------------------------------------------------------------------------

## New Node Arrival

``` text
New node
   |
Hash node identity
   |
Place on ring
   |
Determine newly owned range
   |
Migrate data
   |
Synchronize writes
   |
Verify
   |
Switch routing
```

Important:

> Consistent hashing determines ownership; a separate
> migration/rebalancing mechanism moves the physical data.

------------------------------------------------------------------------

## Virtual Nodes

**Question:** Why use virtual nodes?

**Answer:**

A single position per physical server can create uneven ownership.
Multiple virtual positions per physical server distribute ownership more
evenly and spread rebalancing across multiple physical nodes.

------------------------------------------------------------------------

## Cross-Shard Queries

**Question:** How do you handle them?

Answer in this order:

``` text
1. Try to target one shard using the shard key.
2. Co-locate frequently joined data.
3. Use scatter-gather when unavoidable.
4. Merge/aggregate results at the router/service layer.
5. Avoid cross-shard transactions where possible.
6. Control fan-out with limits, timeouts and bounded concurrency.
```

------------------------------------------------------------------------

# 51. Very Important Interview Distinctions

### Consistent Hashing

``` text
Mapping problem
```

> Which shard should own this key?

### Virtual Nodes

``` text
Distribution problem
```

> How do we balance ownership across physical servers?

### Rebalancing

``` text
Data movement problem
```

> How do we physically move records after ownership changes?

### Cross-Shard Query

``` text
Query routing problem
```

> What if the query cannot identify one shard?

### Cross-Shard Transaction

``` text
Distributed consistency problem
```

> What if one business transaction touches multiple shards?

------------------------------------------------------------------------

# 52. Final Mental Model

Keep this complete picture in your head:

``` text
                         APPLICATION
                              |
                              v
                       SHARD ROUTER
                              |
                    hash(partition key)
                              |
                              v
                        HASH RING
                              |
                +-------------+-------------+
                |             |             |
              VNode         VNode         VNode
                |             |             |
               DB1           DB2           DB3
                |
                |
        +-------+-------+
        |               |
    Physical         Physical
      Data             Data
```

When a node joins:

``` text
New Node
   |
   v
Add to ring
   |
   v
New ownership range
   |
   v
Migrate affected data
   |
   v
Synchronize changes
   |
   v
Switch routing
```

When a query arrives:

``` text
Query
  |
  +-- Contains shard key?
  |       |
  |      YES
  |       |
  |       v
  |   One shard
  |
  +-- NO
      |
      v
  Scatter-Gather
      |
      v
 Multiple shards
      |
      v
 Merge / Aggregate
```

When a node fails:

``` text
Node fails
   |
   v
Its virtual nodes disappear
   |
   v
Ownership ranges move
   |
   v
Surviving nodes take ranges
```

------------------------------------------------------------------------

# 53. One-Minute Bloomberg Interview Explanation

If asked to explain the overall approach:

> "For horizontal database scaling, I'd choose a suitable shard key
> based on the primary access patterns. I'd use consistent hashing so
> keys map onto a logical hash ring and a key is assigned to the first
> shard clockwise. This avoids the large-scale remapping we'd get from
> `hash(key) % N` when the number of shards changes. I'd use virtual
> nodes so each physical shard owns many positions on the ring,
> improving distribution and making node failures or additions spread
> their impact across multiple servers. When a node joins, only the
> ranges it takes ownership of need to be migrated, and I'd use a
> controlled migration process with synchronization and verification
> before switching routing. For queries that contain the shard key, I'd
> target a single shard. For unavoidable cross-shard queries, I'd use
> parallel scatter-gather and merge the results, while designing the
> shard key and data model to minimize cross-shard joins and
> transactions."

------------------------------------------------------------------------

# 54. Memory Trick

Remember these five lines:

``` text
SHARDING
→ Split data across machines.

CONSISTENT HASHING
→ Minimize data remapping when machines change.

VIRTUAL NODES
→ Balance ownership across machines.

REBALANCING
→ Physically move data after ownership changes.

CROSS-SHARD QUERY
→ Query multiple machines and merge results.
```
