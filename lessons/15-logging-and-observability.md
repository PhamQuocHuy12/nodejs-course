# Lesson 15 - Logging and Observability

> Reading time: about 28 minutes  
> Practice time: about 60 minutes

## Outcome

Produce structured logs, connect events with request IDs, measure request duration, distinguish logs from metrics and traces, and design meaningful health checks.

## Observability asks questions of a running system

Tests tell you what happened in controlled scenarios. Observability helps answer unexpected production questions:

- Which requests are failing?
- When did latency increase?
- Is the database unavailable?
- Did one user action cross several services?
- Is memory growing?

Three common signals:

- Logs: discrete event records
- Metrics: numeric measurements over time
- Traces: a request path across operations and services

## 1. Prefer structured logs

Hard-to-query text:

~~~javascript
console.log("User 42 created session 81");
~~~

Structured event:

~~~javascript
console.log(JSON.stringify({
  level: "info",
  event: "study_session_created",
  userId: 42,
  sessionId: 81,
}));
~~~

Machines can reliably search fields. Humans can still read them.

## 2. Build a tiny logger

~~~javascript
const levelPriority = {
  debug: 10,
  info: 20,
  warn: 30,
  error: 40,
};

export function createLogger(minimumLevel = "info") {
  function write(level, event, fields = {}) {
    if (levelPriority[level] < levelPriority[minimumLevel]) {
      return;
    }

    const record = {
      timestamp: new Date().toISOString(),
      level,
      event,
      ...fields,
    };

    const output = JSON.stringify(record);

    if (level === "error") console.error(output);
    else console.log(output);
  }

  return {
    debug: (event, fields) => write("debug", event, fields),
    info: (event, fields) => write("info", event, fields),
    warn: (event, fields) => write("warn", event, fields),
    error: (event, fields) => write("error", event, fields),
  };
}
~~~

Production projects commonly use a mature structured logger, but writing this version makes the data model visible.

## 3. Add a request identifier

~~~javascript
import { randomUUID } from "node:crypto";

export function requestContext(request, response, next) {
  request.requestId = request.get("x-request-id") ?? randomUUID();
  response.set("x-request-id", request.requestId);
  next();
}
~~~

A client-supplied ID can aid tracing, but validate its length and format before trusting it. For public systems, generating your own internal ID is often safer.

## 4. Log request completion

~~~javascript
export function requestLogger(logger) {
  return function logRequest(request, response, next) {
    const startedAt = process.hrtime.bigint();

    response.on("finish", () => {
      const elapsedNanoseconds =
        process.hrtime.bigint() - startedAt;

      logger.info("http_request_completed", {
        requestId: request.requestId,
        method: request.method,
        path: request.path,
        statusCode: response.statusCode,
        durationMs: Number(elapsedNanoseconds) / 1_000_000,
      });
    });

    next();
  };
}
~~~

Log when the response finishes so status and duration are known. Avoid logging query strings blindly because they can contain private data.

## 5. Log errors once with useful context

The central error handler is a natural boundary:

~~~javascript
logger.error("request_failed", {
  requestId: request.requestId,
  errorName: error.name,
  errorMessage: error.message,
  errorCode: error.code,
  stack: error.stack,
});
~~~

Client responses should hide unexpected internals. Internal logs can preserve them. Avoid logging the same failure at every layer because duplicates obscure the event.

## 6. Redact sensitive data

Never log:

- Passwords or password hashes
- Authorization headers
- Cookies and raw session tokens
- Database connection strings with credentials
- Full payment or identity data
- Unbounded request bodies

A request ID lets you correlate activity without copying secrets into every line.

## 7. Health and readiness are different

Liveness asks:

> Is the process running well enough that restarting it might help?

Readiness asks:

> Can this instance currently serve traffic?

~~~javascript
app.get("/health", (request, response) => {
  response.status(200).json({ status: "ok" });
});

app.get("/ready", async (request, response) => {
  try {
    await dataSource.query("SELECT 1");
    response.status(200).json({ status: "ready" });
  } catch {
    response.status(503).json({ status: "not_ready" });
  }
});
~~~

Do not make liveness depend on every downstream service. A database outage should not necessarily cause an endless process restart loop.

## 8. Basic runtime measurements

~~~javascript
const memory = process.memoryUsage();

logger.debug("process_memory", {
  rssBytes: memory.rss,
  heapUsedBytes: memory.heapUsed,
  heapTotalBytes: memory.heapTotal,
});
~~~

One sample proves little. Metrics become useful as trends, rates, and distributions with meaningful labels.

Avoid high-cardinality metric labels such as raw user IDs or request IDs.

## 9. Context across async work

Node.js AsyncLocalStorage can carry request context through asynchronous calls without passing requestId into every function. It is useful but adds invisible state, so introduce it only after understanding explicit context flow.

## Real-world applications and edge cases

### Where this appears

- On-call engineers trace one failed request across a proxy and several services.
- Product teams measure latency and failure rates for important user journeys.
- Security teams audit authentication and privilege changes.
- Operators decide whether a deployment should continue or roll back.

### Edge cases to investigate

- A log field contains unbounded user input and dramatically increases storage cost.
- User IDs, URLs, or exception messages become high-cardinality metric labels.
- Aggressive sampling removes the one rare failure needed during an incident.
- Several layers log the same error and make one failure look like five.
- Async context is lost across an unusual callback boundary, so the request ID disappears.
- Servers have clock skew and log events in misleading order.
- A synchronous logger blocks the event loop under heavy output.
- Buffered logs are lost when the process crashes or is forcibly terminated.
- Readiness checks overload the database they are trying to measure.
- Personal data remains in logs longer than application data-retention policy allows.

Observability data is production data: define schemas, ownership, retention, privacy, sampling, and failure behavior.

## Pause and predict

- Why log on the response finish event?
- Should a failed database readiness check make liveness fail?
- Why is request ID useful but unsuitable as a metric label?
- Should a client receive the unexpected error stack?

## Guided practice

Add:

- Validated request IDs
- JSON request-completion logs
- Central error logs
- Secret redaction tests
- GET /health
- GET /ready with a database check and short timeout
- Log-level configuration
- Rate-limit rejection counts without logging sensitive key material
- Shared limiter-store availability

Trigger invalid input, authentication failure, missing data, and database failure. Compare their logs.

## Common mistakes

- Writing unstructured prose only
- Logging secrets or entire request objects
- Logging the same error at every layer
- Using error level for ordinary client validation failures
- Omitting request IDs
- Treating health and readiness as synonyms
- Adding user IDs or request IDs as metric labels
- Measuring without deciding which question the measurement answers

## Independent challenge

Simulate a slow repository call. Add a warning when a request exceeds a chosen duration, then write an incident note containing:

- User-visible symptom
- Time window
- Affected route
- Evidence from logs
- Likely cause
- Next diagnostic step

Do not claim a cause that the evidence cannot support.

## Knowledge check

- How do logs, metrics, and traces differ?
- Why are structured fields useful?
- What belongs in an error log but not the response?
- What do liveness and readiness each answer?
- Why does cardinality matter for metrics?

## Explore with an AI agent

- Inspect these logs and separate evidence from inference.
- Review my log fields for missing context and accidental secret exposure.
- Help me design three useful API metrics with low-cardinality labels.
- Explain AsyncLocalStorage by tracing one request through nested async functions.

## Official reading

- [Diagnostics overview](https://nodejs.org/en/learn/diagnostics)
- [Diagnostics channel API](https://nodejs.org/api/diagnostics_channel.html)
- [Performance hooks](https://nodejs.org/api/perf_hooks.html)
- [Process memory usage](https://nodejs.org/api/process.html#processmemoryusage)
- [Async context tracking](https://nodejs.org/api/async_context.html)

## Definition of done

- Logs are structured and correlated by request ID.
- Sensitive fields are absent or redacted.
- Health and readiness reflect different states.
- Unexpected errors are logged internally without leaking to clients.
