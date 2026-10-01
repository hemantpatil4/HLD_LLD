# Bloomberg URL Shortener — Load Balancer Deep Dive

## 1. Why do we need a Load Balancer?

Suppose we have only one API server:

```text
Users
  |
  v
API Server
  |
 Redis
  |
  DB
```

If traffic increases, one server becomes a bottleneck.

Instead:

```text
                    Users
                      |
                      v
                Load Balancer
                 /     |     \
                v      v      v
              API1   API2   API3
                 \     |     /
                    Redis
                      |
                      DB
```

The Load Balancer distributes incoming requests across multiple API instances.

Core idea:

> The Load Balancer provides a single entry point while distributing traffic across healthy backend instances.

---

## 2. Why horizontal scaling needs a Load Balancer

Suppose one API server can handle:

```text
50,000 requests/sec
```

and traffic becomes:

```text
150,000 requests/sec
```

We can run:

```text
API1 -> 50K
API2 -> 50K
API3 -> 50K
```

The Load Balancer distributes requests.

This is horizontal scaling:

```text
1 server
   |
   +---- add server
   |
   +---- add server
```

Instead of making one machine increasingly powerful.

---

## 3. What does the Load Balancer actually do?

Conceptually:

```text
Request
   |
   v
Load Balancer
   |
   +----> API1
   |
   +----> API2
   |
   +----> API3
```

It typically performs:

- Request distribution
- Health checking
- Connection management
- TLS termination in some architectures
- Routing
- Failure detection
- Sometimes rate limiting/WAF integration

---

## 4. Health checks

Suppose:

```text
API1 -> Healthy
API2 -> Healthy
API3 -> Down
```

The Load Balancer should stop sending new requests to API3.

```text
              Load Balancer
             /       |       \
            v        v        X
          API1      API2     API3
         healthy   healthy    down
```

Typical endpoint:

```http
GET /health
```

Response:

```http
200 OK
```

A more useful production design can distinguish:

```text
/health/live
/health/ready
```

### Liveness

"Is the process alive?"

### Readiness

"Can this instance currently receive traffic?"

For example, an API whose critical dependencies are unavailable may be marked not ready.

---

## 5. Load-balancing algorithms

### Round Robin

Requests are distributed sequentially:

```text
Request 1 -> API1
Request 2 -> API2
Request 3 -> API3
Request 4 -> API1
```

Simple and common.

### Least Connections

Send the request to the server with the fewest active connections.

```text
API1 -> 100 connections
API2 -> 30 connections
API3 -> 60 connections

Next request -> API2
```

Useful when request durations vary.

### Weighted Load Balancing

Suppose:

```text
API1 -> weight 2
API2 -> weight 1
```

API1 receives roughly twice as much traffic.

Useful when servers have different capacities.

### IP Hash / Consistent Routing

Requests can be mapped based on a key such as client IP.

This can provide affinity, but sticky sessions should not be required for a properly stateless API.

---

## 6. Stateless API

A URL shortener API should ideally be stateless.

Bad design:

```text
API1
  |
local memory:
"user session -> data"
```

Then:

```text
Request 1 -> API1
Request 2 -> API2
```

API2 does not have API1's local state.

Better:

```text
API1 \
API2  ---> Redis / DB
API3 /
```

Shared state lives in shared infrastructure.

Therefore:

```text
Request 1 -> API1
Request 2 -> API3
Request 3 -> API2
```

All can work.

---

## 7. Connection draining

Suppose API2 is being removed for deployment.

We do not want:

```text
Request -> API2
              |
              X shutdown
```

Instead:

```text
Load Balancer
     |
     X stop new requests
     |
API2 finishes existing requests
     |
     v
shutdown
```

This is connection draining / graceful shutdown.

---

## 8. Failure example

Suppose:

```text
API1 -> healthy
API2 -> healthy
API3 -> failed
```

The Load Balancer detects API3 failure.

Traffic becomes:

```text
          Load Balancer
           /          \
          v            v
        API1          API2
```

The system continues serving requests if remaining capacity is sufficient.

---

## 9. Load Balancer is not the whole scaling solution

A common interview mistake:

> "We'll add a Load Balancer and the system will scale."

Not necessarily.

Consider:

```text
Users
  |
Load Balancer
  |
APIs
  |
Redis
  |
Database
```

If APIs scale from 3 to 30 instances:

```text
30 APIs
   |
   v
Redis
   |
   v
Database
```

Redis or DB may now become the bottleneck.

So every layer needs capacity planning.

---

## 10. URL Shortener architecture

```text
                           Users
                             |
                             v
                       Load Balancer
                    /       |       \
                   v        v        v
                 API1      API2     API3
                   \        |        /
                    \       |       /
                         Redis
                           |
                    Cache miss only
                           |
                           v
                    Read Replicas
                           |
                           v
                       Primary DB
```

Write:

```text
POST
 |
Load Balancer
 |
API
 |
Primary DB
 |
ID
 |
Base62
 |
Redis
 |
Response
```

Read:

```text
GET /aZ91k
 |
Load Balancer
 |
API
 |
Redis HIT
 |
302
```

---

## 11. Interview answer

If asked "Why do you need a Load Balancer?":

> "Because I want to horizontally scale the stateless API layer. The Load Balancer gives clients a single endpoint and distributes requests across healthy API instances. It can perform health checks and remove unhealthy instances from rotation. Since the API is stateless and state is in Redis and the database, requests can go to any healthy instance."

---

## 12. Common interview follow-ups

### What if one API server dies?

The Load Balancer health check detects it and stops routing new requests there.

### Do we need sticky sessions?

Not if the API is properly stateless.

### Can the Load Balancer become a bottleneck?

A production Load Balancer is itself deployed redundantly/scalably, often as managed infrastructure.

### Does a Load Balancer solve database scaling?

No.

It primarily distributes traffic at the service/connection layer. Database scaling is a separate concern.

---

## 13. Key takeaway

```text
Load Balancer
     |
     v
Horizontal API scaling
     |
     v
Stateless API instances
     |
     v
Shared Redis + Database
```

Remember:

> Load Balancer distributes traffic. It does not magically scale every downstream dependency.
