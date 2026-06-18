# Lesson 13 - Rate Limiting and API Protection

> Reading time: about 35 minutes  
> Practice time: about 80 minutes

## Outcome

Design rate limits from actual abuse and capacity risks, return useful HTTP responses, implement route-specific limits in Express, and explain why multiple application instances require shared limiter state.

## Why rate limiting matters

Authentication answers who a caller is. Authorization answers what that caller may access. Neither prevents a permitted caller from making far too many requests.

Rate limiting can reduce:

- Password guessing
- Automated account creation
- Expensive report abuse
- Accidental retry storms
- Resource exhaustion
- Unfair consumption by one client

It is one protective layer. It does not replace validation, authorization, timeouts, queue bounds, or capacity planning.

## 1. A limit needs four decisions

Every rate-limit policy must define:

1. Key: who or what is counted?
2. Cost: how much does this action consume?
3. Capacity: how much is allowed?
4. Time behavior: how is capacity restored?

Example:

~~~text
Key: authenticated user ID
Cost: one token per report request
Capacity: five requests
Time behavior: restore five tokens every hour
~~~

“Add rate limiting” is not yet a policy.

## 2. Choose a key carefully

Possible keys:

- Client IP address
- Authenticated user ID
- API key
- Account plus route
- Organization or tenant ID
- A combination of user and IP

IP limits help before authentication, but they have limitations:

- Many users may share one network address.
- One attacker may use many addresses.
- IPv6 clients may rotate addresses inside a prefix.
- A reverse proxy changes which address Express sees.

After authentication, user or tenant identity is usually more meaningful.

## 3. Fixed-window counting

A fixed window counts requests in periods such as each minute.

~~~text
12:00:00 through 12:00:59 -> maximum 100
12:01:00 through 12:01:59 -> maximum 100
~~~

It is easy to implement but has a boundary burst: a client can send 100 requests near the end of one minute and 100 more at the start of the next.

### Simple mental model

~~~javascript
const key = clientId + ":" + currentMinute;
const count = await store.increment(key);

if (count > 100) {
  throw new RateLimitError();
}
~~~

This is illustrative. A real shared store must increment and set expiration atomically.

## 4. Sliding-window approaches

A sliding log records individual request times and counts only those inside the current interval. It is accurate but stores more data.

A sliding-window counter approximates the result using neighboring fixed windows and weighted counts. It reduces boundary bursts with less storage than a complete log.

Choose complexity from the threat and traffic pattern. Login protection and ordinary read traffic do not necessarily need the same algorithm.

## 5. Token bucket

A token bucket has:

- Maximum token capacity
- Refill rate
- Cost per operation

A request spends tokens. Unused capacity accumulates up to the maximum, allowing a bounded burst.

~~~text
Capacity: 20 tokens
Refill: 2 tokens per second
Normal request: 1 token
Report generation: 5 tokens
~~~

Token buckets fit APIs where short bursts are acceptable but long-term rate must remain controlled.

## 6. Respond with HTTP 429

When a client exceeds policy, return:

~~~text
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json
~~~

~~~json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Try again later."
  }
}
~~~

Standard rate-limit fields can communicate policy and remaining capacity. Follow the format supported by your limiter and clients. Do not invent misleading reset values.

Clients should use bounded retries with backoff and jitter rather than immediately retrying every rejected request.

## 7. Add Express rate limiting

Install:

~~~powershell
npm install express-rate-limit
~~~

Create a general limiter:

~~~javascript
import { rateLimit } from "express-rate-limit";

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 100,
  standardHeaders: "draft-8",
  legacyHeaders: false,
  handler(request, response) {
    response.status(429).json({
      error: {
        code: "RATE_LIMIT_EXCEEDED",
        message: "Too many requests. Try again later.",
      },
    });
  },
});
~~~

Register it before protected routes:

~~~javascript
app.use("/api", apiLimiter);
~~~

Version-specific options can change. Check the linked project documentation when installing.

## 8. Sensitive routes need separate policies

Login and registration deserve stricter policies than ordinary reads:

~~~javascript
export const loginLimiter = rateLimit({
  windowMs: 10 * 60 * 1000,
  limit: 10,
  standardHeaders: "draft-8",
  legacyHeaders: false,
  skipSuccessfulRequests: true,
});
~~~

Apply:

~~~javascript
app.post(
  "/auth/login",
  loginLimiter,
  authController.login,
);
~~~

Skipping successful requests can focus the count on failed attempts, but account enumeration and distributed guessing still require thought. Combine per-IP and per-account controls without revealing whether the account exists.

## 9. Use authenticated identity after authentication

Middleware order matters:

~~~javascript
app.get(
  "/reports/course-totals",
  authenticate,
  reportLimiter,
  reportController.courseTotals,
);
~~~

The limiter can now key by request.user.id:

~~~javascript
const reportLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  limit: 5,
  keyGenerator(request) {
    return "user:" + request.user.id;
  },
});
~~~

For unauthenticated routes, use a carefully configured network identity or a composite key.

## 10. Reverse proxies change client addresses

Behind a reverse proxy or load balancer, request.ip depends on Express proxy configuration.

~~~javascript
app.set("trust proxy", 1);
~~~

The value 1 is correct only for a known one-hop topology. Trusting arbitrary forwarded headers can let clients spoof their address and bypass or weaponize IP limits.

Document the exact proxy chain, then configure the narrowest correct trust rule. Lesson 19 develops this further.

## 11. One process versus many processes

The default memory store lives in one Node.js process:

~~~text
Instance A count for user 42: 8
Instance B count for user 42: 7
~~~

If the intended limit is 10, the user has already made 15 requests without either instance observing the global total.

Horizontal scaling requires a shared store such as Redis or another service with atomic update and expiry behavior.

The application should decide what happens if that store fails:

- Fail open: allow traffic and preserve availability.
- Fail closed: reject traffic and preserve strict protection.
- Degraded policy: use a conservative local fallback.

Login and payment-like operations may choose differently from public reads.

## 12. Rate is not the only resource control

Ten requests per minute can still be dangerous if all ten run expensive work simultaneously.

Also consider:

- Concurrency limits
- Bounded work queues
- Request timeouts
- Database statement timeouts
- Body-size limits
- Per-operation cost
- Circuit breakers for unhealthy dependencies

Rate limiting controls arrival over time. A concurrency limit controls work happening at once.

## 13. WebSocket protection

Lesson 17 applies related controls to long-lived connections:

- Connection attempts per IP or user
- Maximum active connections per user
- Messages per second
- Message size
- Expensive subscription count
- Slow-consumer handling

An HTTP-only limiter stops seeing traffic after the WebSocket upgrade, so message-level protection must live in the real-time layer.

## Real-world applications and edge cases

### Where this appears

- Login endpoints slow password guessing.
- Public search endpoints protect expensive database queries.
- Report generation uses weighted quotas and concurrency limits.
- A SaaS product enforces per-tenant plan limits.
- Webhook receivers protect themselves from retry storms.

### Edge cases to investigate

- Thousands of legitimate users share one corporate or carrier-grade NAT address.
- An attacker rotates IP addresses but targets one account.
- A fixed-window boundary permits twice the intended short burst.
- Redis becomes unavailable and different routes need different fail-open or fail-closed behavior.
- A multi-region deployment has counters that are not immediately consistent.
- Successful login decrement logic races with concurrent failed attempts.
- A request is rejected at the CDN before application-level headers or logs are produced.
- A client retries every 429 immediately and makes overload worse.
- One cheap request and one costly export consume the same quota despite very different resource use.
- Proxy misconfiguration lets clients choose the apparent IP key.

Observe rejected traffic, allowed traffic, store errors, and user impact. A limit that cannot be explained during an incident is difficult to operate safely.

## Pause and predict

- Why can an IP-only login limit block innocent users?
- Why does a memory limiter become inaccurate with two instances?
- Which should be stricter: a health endpoint or password login?
- Does 100 requests per minute prevent 100 concurrent expensive reports?
- When is fail closed safer than fail open?

## Guided practice

Add three policies to the Study Tracker API:

1. General API limit by authenticated user when available.
2. Strict login limit using both network and account-aware thinking.
3. Expensive report limit by user ID.

For each policy, document:

| Decision | Answer |
|---|---|
| Protected risk | |
| Key | |
| Algorithm | |
| Capacity and window | |
| Response | |
| Shared store required | |
| Store-failure policy | |

Write tests that make requests just below, exactly at, and above each threshold.

## Common mistakes

- Applying one arbitrary limit to every route
- Trusting forwarded IP headers without proxy configuration
- Using only in-memory state across several instances
- Counting successful and failed login attempts without deliberate policy
- Returning 429 without retry information
- Allowing unlimited concurrent expensive work
- Treating rate limiting as protection from every denial-of-service attack
- Using user-controlled key values without normalization or bounds
- Forgetting WebSocket message and connection limits

## Independent challenge

Design a distributed token-bucket policy for report generation:

- Five tokens per user
- One token restored every 12 minutes
- Normal report costs one token
- CSV report costs three tokens
- Maximum two reports running per user

Describe the atomic shared-store operation, response fields, concurrency interaction, and behavior when the store is unavailable. Then implement the policy behind an interface so tests can use a deterministic fake clock and store.

## Knowledge check

- What four decisions define a limit?
- How do fixed-window and token-bucket behavior differ?
- Why is authenticated identity often better than IP?
- What does HTTP 429 communicate?
- Why must distributed instances share counters?
- How does rate limiting differ from concurrency limiting?

## Explore with an AI agent

- Threat-model my API and propose separate limits based on evidence, not generic numbers.
- Review my Express middleware order for identity and proxy mistakes.
- Simulate a fixed-window boundary burst and compare it with a token bucket.
- Challenge my fail-open or fail-closed choice for each endpoint.
- Generate deterministic tests using a fake clock without writing the implementation.

## Official reading

- [Express rate limit overview](https://express-rate-limit.mintlify.app/overview)
- [Express rate limit project](https://github.com/express-rate-limit/express-rate-limit)
- [HTTP 429 status](https://www.rfc-editor.org/rfc/rfc6585.html#section-4)
- [RateLimit header fields](https://www.rfc-editor.org/rfc/rfc9333.html)
- [Express behind proxies](https://expressjs.com/en/guide/behind-proxies.html)
- [Express production security](https://expressjs.com/en/advanced/best-practice-security.html)

## Definition of done

- Each limit corresponds to a documented risk.
- Login, general traffic, and expensive routes have separate policies.
- Responses use HTTP 429 and useful standard fields.
- Proxy-derived identity is configured safely.
- The production design uses shared state across instances.
- Rate and concurrency protections are tested independently.
