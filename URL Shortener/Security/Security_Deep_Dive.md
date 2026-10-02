# Bloomberg URL Shortener — Security Deep Dive

## 1. What are we protecting?

A URL shortener accepts user-controlled input and exposes public endpoints.

Threats include:

```text
Malicious URLs
Abuse
Spam
Phishing
Open redirects
Enumeration
DDoS
Credential/token theft
Injection
Unauthorized administration
```

Security must be part of the system design.

---

## 2. Validate the original URL

Client sends:

```json
{
  "url": "https://example.com"
}
```

The API should validate:

- Valid URL syntax
- Allowed schemes
- Maximum URL length
- Normalization rules
- Potentially allowed/blocked domains depending on product requirements

For a typical public shortener:

```text
https://example.com
```

may be allowed.

Potentially dangerous schemes such as:

```text
javascript:
data:
```

should not be accepted as normal HTTP URLs.

---

## 3. Avoid open redirect abuse

The redirect endpoint:

```text
GET /aZ91k
```

should only redirect to a URL stored in the database.

Do not design it as:

```text
GET /redirect?url=<arbitrary-user-input>
```

That can become an open redirect service.

Better:

```text
Short Code
    |
    v
Database
    |
    v
Stored Original URL
    |
    v
302
```

---

## 4. Rate limiting

Without rate limiting:

```text
Attacker
   |
   +--> 1M requests/sec
   |
   v
API
   |
   v
Redis / DB
```

Potential overload.

Apply rate limits by appropriate dimensions:

```text
IP
User
API key
Endpoint
Tenant
```

Example:

```text
POST /api/urls
100 requests/minute/user
```

The exact limits depend on the product.

---

## 5. Authentication and authorization

Creating URLs might require authentication depending on the product.

Example:

```text
POST /api/urls
   |
Authentication
   |
Authorization
   |
Create URL
```

Authentication answers:

> Who are you?

Authorization answers:

> What are you allowed to do?

Public redirect:

```text
GET /aZ91k
```

may intentionally remain unauthenticated.

Administrative operations should require authorization.

---

## 6. HTTPS

Use TLS:

```text
Client
  |
 HTTPS
  |
Load Balancer
```

Protects data in transit.

Internal service communication should also be protected according to the environment and threat model.

---

## 7. Secrets

Never hard-code:

```csharp
var password = "MyPassword123";
```

Use appropriate secret management.

Examples:

```text
Environment / secret store
Managed identity
Cloud secret manager
Vault
```

The exact mechanism depends on infrastructure.

---

## 8. SQL injection

Do not build SQL using string concatenation.

Bad:

```csharp
var sql = "SELECT * FROM UrlMappings WHERE Id = " + id;
```

Prefer parameterized queries or EF Core:

```csharp
var mapping = await db.UrlMappings
    .FirstOrDefaultAsync(x => x.Id == id);
```

The ORM generates parameterized SQL.

---

## 9. Enumeration

If IDs are sequential:

```text
/21
/22
/23
/24
```

users can infer that identifiers are sequential.

This may expose information about:

- Approximate creation volume
- Ordering
- Number of records

Using a distributed ID plus Base62 can make the public identifier less obvious, although Base62 itself is not encryption.

If confidentiality of identifiers is required, use an appropriate non-reversible public identifier design.

---

## 10. Abuse prevention

A public URL shortener can be abused for:

```text
Spam
Phishing
Malware distribution
Link obfuscation
Automated attacks
```

Possible controls:

```text
Rate limiting
Domain reputation checks
Abuse reporting
Blocklists
Allow lists where appropriate
URL scanning
User/account controls
CAPTCHA or challenge mechanisms
```

These are product/security decisions and may vary by deployment.

---

## 11. DDoS

A public shortener can receive huge request volumes.

Layers of defense:

```text
Internet
   |
DDoS protection / CDN / WAF
   |
Load Balancer
   |
API
   |
Redis
   |
DB
```

The objective is to prevent malicious traffic from reaching expensive backend resources.

---

## 12. Cache security

Be careful about what enters Redis.

Bad:

```text
User-controlled arbitrary object
```

Better:

```text
Validated short code
    |
    v
Namespaced key
url:aZ91k
```

Also protect Redis itself:

- Network isolation
- Authentication/authorization where supported
- TLS where required
- No unnecessary public exposure

---

## 13. Security headers and response behavior

Depending on the application, consider appropriate HTTP security headers and avoid leaking internal information through error responses.

Do not return:

```text
SQL exception
Stack trace
Connection string
Internal server path
```

to public clients.

---

## 14. Data protection

Protect:

```text
Database
Redis
Logs
Backups
Secrets
Administrative endpoints
```

Apply least privilege.

For example:

```text
API service
   |
   +--> DB: required operations only
   |
   +--> Redis: required operations only
```

Avoid giving the application unrestricted database permissions.

---

## 15. Interview answer

> "For security I'd validate and normalize submitted URLs, restrict dangerous schemes, prevent arbitrary open redirects, add rate limiting and abuse controls, use HTTPS, authenticate and authorize management APIs, parameterize database access, protect Redis and secrets, and put public traffic behind DDoS/WAF controls where appropriate. I would also avoid exposing internal errors and use least-privilege access."

---

## 16. Key takeaway

Security is layered:

```text
Client
  |
DDoS / WAF
  |
HTTPS
  |
Load Balancer
  |
Authentication / Rate Limiting
  |
API validation
  |
Redis / DB
  |
Least privilege + monitoring
```
