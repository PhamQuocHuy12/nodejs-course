# Lesson 21 - Outbound HTTP and Resilience

> Reading time: about 42 minutes  
> Practice time: about 100 minutes

## Outcome

Call external services with native fetch, enforce deadlines and response limits, reuse connections, retry only safe operations, and contain dependency failures with concurrency and circuit-breaking policies.

## Why outbound calls are a system boundary

An HTTP call leaves your process and depends on:

- DNS
- Network routing
- TLS
- The remote server
- Connection pools
- Proxy infrastructure
- Response parsing

The dependency may be slow, unavailable, overloaded, incorrect, or partially successful. A resilient client makes these states explicit.

## 1. Native fetch

~~~javascript
const response = await fetch(
  "https://example.com/api/courses",
);

if (!response.ok) {
  throw new Error(
    "Course service returned " + response.status,
  );
}

const data = await response.json();
console.log(data);
~~~

Modern Node.js provides fetch globally. It follows the web Request and Response model and is powered by the Undici HTTP client implementation.

## 2. HTTP errors do not reject fetch automatically

fetch rejects for failures such as connection or name-resolution errors. A normal HTTP 404 or 500 still resolves to a Response.

~~~javascript
const response = await fetch(url);

if (response.status === 404) {
  return null;
}

if (!response.ok) {
  throw new UpstreamError(
    "Unexpected upstream status",
    {
      status: response.status,
    },
  );
}
~~~

Translate status according to the dependency contract. Do not turn every non-200 status into the same generic error.

## 3. Response bodies are one-use streams

~~~javascript
const response = await fetch(url);

const text = await response.text();
console.log(text);
~~~

After consuming the body, another attempt to call response.json or response.text fails.

Choose one representation and validate it. A content-type header is useful evidence but does not guarantee valid JSON.

## 4. Apply a deadline

~~~javascript
const response = await fetch(url, {
  signal: AbortSignal.timeout(3_000),
});
~~~

A deadline prevents one dependency from occupying request, socket, and concurrency capacity indefinitely.

The correct timeout depends on:

- Caller deadline
- Normal and tail latency
- Retry budget
- User expectation
- Downstream service objective

Three retries with independent three-second timeouts can consume much more than a three-second overall deadline. Budget the complete operation.

## 5. Propagate caller cancellation

~~~javascript
function createDependencySignal(
  callerSignal,
  timeoutMilliseconds,
) {
  return AbortSignal.any([
    callerSignal,
    AbortSignal.timeout(timeoutMilliseconds),
  ]);
}

const response = await fetch(url, {
  signal: createDependencySignal(
    requestSignal,
    2_000,
  ),
});
~~~

If the caller disconnects or shutdown begins, downstream work should stop when possible.

## 6. Build a focused client module

~~~javascript
export function createCourseClient({
  baseUrl,
  apiToken,
}) {
  const base = new URL(baseUrl);

  return {
    async findCourse(courseId, { signal }) {
      const url = new URL(
        "/courses/" + encodeURIComponent(courseId),
        base,
      );

      const response = await fetch(url, {
        headers: {
          authorization: "Bearer " + apiToken,
          accept: "application/json",
        },
        signal,
      });

      if (response.status === 404) {
        return null;
      }

      if (!response.ok) {
        throw new Error(
          "Course service returned " +
            response.status,
        );
      }

      return response.json();
    },
  };
}
~~~

Centralizing the client provides one place for:

- Base URL validation
- Authentication
- Timeouts
- Response limits
- Error translation
- Metrics
- Retry policy

## 7. Validate the destination

If users control all or part of an outbound URL, the server may be tricked into requesting:

- Loopback services
- Private network addresses
- Cloud metadata endpoints
- Internal administration systems
- Unexpected protocols or ports

This is server-side request forgery.

Prefer a configured base URL and controlled path components:

~~~javascript
const allowedOrigin =
  "https://courses.example.com";

const url = new URL(path, allowedOrigin);

if (url.origin !== allowedOrigin) {
  throw new Error("Outbound origin is not allowed");
}
~~~

URL allowlists alone may not solve DNS rebinding or redirect behavior. Restrict redirects and network egress according to the threat model.

## 8. Limit response size

Content-Length may be absent or dishonest. Enforce the limit while reading.

~~~javascript
async function readBodyWithLimit(
  response,
  maximumBytes,
) {
  if (!response.body) {
    return Buffer.alloc(0);
  }

  const reader = response.body.getReader();
  const chunks = [];
  let totalBytes = 0;

  try {
    while (true) {
      const { done, value } = await reader.read();

      if (done) break;

      totalBytes += value.byteLength;

      if (totalBytes > maximumBytes) {
        await reader.cancel(
          "Response body limit exceeded",
        );
        throw new Error(
          "Upstream response is too large",
        );
      }

      chunks.push(Buffer.from(value));
    }
  } finally {
    reader.releaseLock();
  }

  return Buffer.concat(chunks);
}
~~~

For large legitimate bodies, stream them to the next destination with backpressure rather than buffering up to a giant limit.

## 9. Connection reuse matters

Opening a new TCP and TLS connection for every request adds latency and resource cost. HTTP clients maintain pools of reusable connections.

Pool settings affect:

- Maximum connections per origin
- Queued requests
- Keep-alive lifetime
- Idle connection cleanup
- Pipelining or multiplexing behavior

Unlimited connections can overload the dependency. A tiny pool can become a bottleneck. Measure queue time and active connections.

Advanced Undici configuration should be isolated in infrastructure code and matched to the supported Node.js version.

## 10. DNS and address behavior

DNS results change and may include IPv4 and IPv6 addresses. Long-lived processes and connection pools may keep using existing addresses after DNS changes.

Failures can differ by:

- Network family
- Resolver
- Cache lifetime
- Container or host configuration
- Regional routing

Log the dependency name and failure category, but avoid exposing internal addresses to clients.

## 11. Retry only when justified

Retries are appropriate when:

- Failure is likely transient.
- The operation is safe or idempotent.
- The remaining deadline allows another attempt.
- The retry will not amplify overload.

Safe methods such as GET are usually easier to retry than creation operations.

POST may be retriable only with a dependency-supported idempotency key:

~~~javascript
const response = await fetch(
  "https://payments.example.com/charges",
  {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": operationId,
    },
    body: JSON.stringify(input),
    signal,
  },
);
~~~

The remote service must persist and enforce the key. Sending a header alone creates no guarantee.

## 12. Backoff and jitter

Immediate synchronized retries can worsen an outage.

~~~javascript
function retryDelay(attempt) {
  const base = Math.min(
    200 * 2 ** attempt,
    5_000,
  );

  return base * (0.5 + Math.random());
}
~~~

Exponential backoff increases delay. Jitter spreads clients across time.

Also consider:

- Retry-After
- Maximum attempts
- Overall deadline
- A shared retry budget
- Which statuses are transient
- Whether the body can be sent again

## 13. A bounded retry loop

~~~javascript
import { setTimeout as delay } from "node:timers/promises";

async function fetchWithRetry(
  url,
  {
    signal,
    attempts = 3,
  },
) {
  let lastError;

  for (
    let attempt = 0;
    attempt < attempts;
    attempt += 1
  ) {
    signal.throwIfAborted();

    try {
      const response = await fetch(url, {
        signal,
      });

      if (
        response.ok ||
        ![429, 502, 503, 504].includes(
          response.status,
        )
      ) {
        return response;
      }

      lastError = new Error(
        "Transient status " + response.status,
      );
    } catch (error) {
      if (signal.aborted) throw error;
      lastError = error;
    }

    if (attempt + 1 < attempts) {
      await delay(
        retryDelay(attempt),
        null,
        { signal },
      );
    }
  }

  throw lastError;
}
~~~

This is a teaching example. A production policy should consume or cancel unwanted response bodies, honor Retry-After, classify network errors, and emit attempt metrics.

## 14. Concurrency limits protect dependencies

A rate limit controls arrival over time. A concurrency limit bounds simultaneous calls.

~~~text
Maximum dependency calls in flight: 20
Queue capacity: 100
Queue full: reject or degrade
~~~

Without a bound, one slow dependency can cause requests to accumulate until memory, sockets, or caller deadlines are exhausted.

Separate bulkheads can prevent one unhealthy dependency from consuming every outbound slot.

## 15. Circuit breakers

A circuit breaker tracks dependency outcomes:

- Closed: calls flow normally.
- Open: calls fail quickly for a cooldown period.
- Half-open: a small number of trial calls test recovery.

A breaker can reduce useless pressure during a clear outage. It also adds state and can reject calls after the service has recovered.

Define:

- Which failures count
- Observation window
- Opening threshold
- Cooldown
- Trial-call limit
- Per-instance or shared state
- Fallback behavior

Do not add a breaker before you have timeouts, concurrency bounds, and useful metrics.

## 16. Fallbacks must be honest

Possible fallbacks:

- Cached stale data
- Partial response
- Queued asynchronous work
- Feature unavailable response
- Local default

A fallback is safe only if it preserves the product's correctness expectations. Returning stale exchange rates or permissions can be worse than returning an error.

## Real-world applications and edge cases

### Where this appears

- Payment and identity providers
- Internal microservices
- Email and messaging APIs
- Webhook delivery
- Object storage and media services
- Third-party analytics

### Edge cases to investigate

- The remote server commits a POST but the connection closes before the response.
- DNS succeeds but one returned address is unreachable.
- TLS certificates expire or do not match the expected host.
- The response status is successful but JSON is malformed or semantically invalid.
- A redirect sends credentials to an unexpected origin.
- A response body is never consumed or cancelled, reducing connection reuse.
- Every request retries during an outage and multiplies load.
- One dependency exhausts the global connection pool.
- A timeout fires while the remote service continues the side effect.
- A circuit breaker opens separately in each instance and produces uneven behavior.
- A fallback returns dangerously stale authorization or pricing data.

Treat dependency calls as distributed operations: success, failure, and timeout do not always reveal whether the remote side performed the action.

## Pause and predict

- Does fetch reject automatically for HTTP 500?
- Why must a response body be consumed or cancelled?
- Can a timeout prove the remote side did nothing?
- Which POST requests are safe to retry?
- How do rate and concurrency limits differ?
- When can a fallback be more dangerous than an error?

## Guided practice

Create a client for a fake Course service:

1. Use a configured and validated base URL.
2. Apply caller cancellation and a shorter dependency deadline.
3. Translate 404 separately from unexpected statuses.
4. Limit response size.
5. Validate the returned JSON shape.
6. Add bounded concurrency.
7. Retry only chosen transient GET failures.
8. Add exponential backoff with jitter.
9. Record attempt count, total duration, status, and failure category.
10. Test malformed JSON, slow responses, disconnects, 429, 503, and oversized bodies.

## Common mistakes

- Assuming fetch rejects for non-2xx statuses
- Omitting an overall deadline
- Retrying every method and failure
- Retrying without jitter
- Ignoring Retry-After
- Allowing user-controlled destination URLs
- Buffering unlimited response bodies
- Logging authorization headers or sensitive bodies
- Creating unbounded outbound concurrency
- Returning stale fallback data without product approval
- Treating a timeout as proof that no side effect occurred

## Independent challenge

Build a resilient webhook sender:

- Each event has a stable delivery ID.
- Payloads are signed.
- Calls have an overall deadline and response-size limit.
- Retries use persisted attempt state, backoff, and jitter.
- Only documented statuses retry.
- A maximum attempt count moves the event to a failed state.
- A receiver can deduplicate repeated delivery IDs.
- Process restart does not lose scheduled attempts.

Explain which state belongs in PostgreSQL, which timers may remain local, and how multiple sender instances claim work without duplication.

## Knowledge check

- What failures cause fetch to reject?
- Why centralize dependency clients?
- What is server-side request forgery?
- How do connection pools affect capacity?
- What conditions justify a retry?
- What states does a circuit breaker have?

## Explore with an AI agent

- Review my retry policy for operations that may already have succeeded.
- Threat-model my outbound URL construction for SSRF.
- Give me dependency failures and ask whether to retry, fail, or degrade.
- Analyze whether my timeouts fit within the caller deadline.
- Review my webhook design for duplicate and lost delivery.

## Official reading

- [Using fetch with Node.js](https://nodejs.org/en/learn/getting-started/fetch)
- [Global fetch API](https://nodejs.org/api/globals.html#fetch)
- [Web Streams API](https://nodejs.org/api/webstreams.html)
- [DNS API](https://nodejs.org/api/dns.html)
- [TLS API](https://nodejs.org/api/tls.html)
- [Undici project](https://github.com/nodejs/undici)

## Definition of done

- Every dependency call has a bounded lifetime.
- Status, body size, content, and destination are validated.
- Connection and concurrency capacity are bounded.
- Retries are safe, limited, delayed, and observable.
- Cancellation propagates from the caller.
- Ambiguous side-effect outcomes have an idempotency strategy.
