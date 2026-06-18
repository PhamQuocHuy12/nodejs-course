# Lesson 20 - Events, Async Context, and Cancellation

> Reading time: about 38 minutes  
> Practice time: about 90 minutes

## Outcome

Use EventEmitter safely, carry request context through asynchronous work with AsyncLocalStorage, and propagate cancellation and deadlines with AbortController.

## Why these APIs belong together

Production Node.js applications coordinate work that starts now and finishes later:

- An event announces that something happened.
- Async context identifies which request caused it.
- Cancellation says the result is no longer needed or its deadline has passed.

Without lifecycle design, listeners accumulate, request IDs disappear, and abandoned work continues consuming resources.

## 1. EventEmitter is an in-process event mechanism

~~~javascript
import { EventEmitter } from "node:events";

const events = new EventEmitter();

events.on("session.created", (session) => {
  console.log("Created session", session.id);
});

events.emit("session.created", {
  id: 42,
  minutes: 30,
});
~~~

on registers a listener. emit invokes listeners registered for that event.

EventEmitter is local to one JavaScript process. It is not a replacement for Redis Pub/Sub, a queue, or a durable event stream.

## 2. Listeners run synchronously

~~~javascript
events.on("example", () => {
  console.log("first");
});

events.on("example", () => {
  console.log("second");
});

console.log("before");
events.emit("example");
console.log("after");
~~~

Output:

~~~text
before
first
second
after
~~~

Listeners run in registration order during emit. A CPU-heavy listener blocks the emitter and the event loop.

To schedule later work deliberately:

~~~javascript
events.on("report.requested", (report) => {
  setImmediate(() => {
    startReport(report);
  });
});
~~~

Scheduling later does not create durability, retries, or queue bounds.

## 3. on, once, and off

on keeps the listener until removed:

~~~javascript
function handleSession(session) {
  console.log(session.id);
}

events.on("session.created", handleSession);
events.off("session.created", handleSession);
~~~

once removes itself after its first invocation:

~~~javascript
events.once("connected", () => {
  console.log("Connected for the first time");
});
~~~

Removing a listener requires the same function reference. An anonymous function cannot later be reconstructed for off.

## 4. The error event is special

If an EventEmitter emits error without an error listener, Node.js throws the error and the process may terminate.

~~~javascript
events.on("error", (error) => {
  console.error("Event failure:", error.message);
});

events.emit("error", new Error("Subscription failed"));
~~~

Use error for actual emitter failures, not as a generic replacement for every domain failure.

## 5. Async listeners need explicit failure handling

~~~javascript
events.on("session.created", async (session) => {
  await sendNotification(session);
});
~~~

emit does not await the returned promise. A rejection can become unhandled.

Handle it explicitly:

~~~javascript
events.on("session.created", (session) => {
  sendNotification(session).catch((error) => {
    events.emit("error", error);
  });
});
~~~

EventEmitter also supports rejection-capture behavior, but explicit application boundaries are easier to reason about while learning.

If notification delivery must survive a crash, use a durable job or event mechanism rather than an in-memory listener.

## 6. Listener warnings are useful evidence

EventEmitter warns when many listeners are attached to one event because accidental listener accumulation is a common memory leak.

Do not silence the warning by immediately increasing the maximum. First ask:

- Who adds the listener?
- Who removes it?
- Is one listener added for every request?
- Does a reconnect path add duplicates?
- Does cleanup run after errors and cancellation?

A higher legitimate listener count should be documented and measured.

## 7. AsyncLocalStorage carries request context

Passing requestId through every function is explicit but repetitive. AsyncLocalStorage can associate a store with one asynchronous execution path.

~~~javascript
import { AsyncLocalStorage } from "node:async_hooks";

export const requestContext = new AsyncLocalStorage();

export function contextMiddleware(
  request,
  response,
  next,
) {
  const context = {
    requestId: request.requestId,
    userId: request.user?.id,
  };

  requestContext.run(context, next);
}
~~~

Read it deeper in the call chain:

~~~javascript
export function currentContext() {
  return requestContext.getStore() ?? {};
}

export function logMessage(message) {
  const context = currentContext();

  console.log(JSON.stringify({
    message,
    requestId: context.requestId,
    userId: context.userId,
  }));
}
~~~

The service no longer needs Express request objects merely to access correlation data.

## 8. Async context is not authorization

Context is convenient metadata, not trusted access control by itself.

Business functions should receive important identity explicitly when it affects correctness:

~~~javascript
await courseService.update({
  userId: request.user.id,
  courseId: request.params.id,
  changes: request.body,
});
~~~

Use AsyncLocalStorage for tracing, logging, and diagnostics. Avoid making critical permission checks depend on invisible mutable state.

## 9. AbortController represents cancellation

~~~javascript
const controller = new AbortController();
const { signal } = controller;

controller.abort(
  new Error("The operation is no longer needed"),
);

console.log(signal.aborted);
console.log(signal.reason);
~~~

The signal announces cancellation. APIs must cooperate with it. AbortController cannot forcibly rewind arbitrary JavaScript or undo a committed database transaction.

## 10. Cancel a promise-based timer

~~~javascript
import { setTimeout as delay } from "node:timers/promises";

const controller = new AbortController();

setTimeout(() => {
  controller.abort(new Error("Deadline exceeded"));
}, 100);

try {
  await delay(1_000, null, {
    signal: controller.signal,
  });
} catch (error) {
  if (controller.signal.aborted) {
    console.error(controller.signal.reason.message);
  } else {
    throw error;
  }
}
~~~

The timer releases its work after cancellation instead of waiting for the full second.

## 11. Deadlines are cancellation with time

~~~javascript
const signal = AbortSignal.timeout(3_000);

const response = await fetch(
  "https://example.com/data",
  { signal },
);
~~~

A deadline limits how long the caller is willing to wait. Every downstream operation should receive a deadline no later than the remaining request deadline.

Version-specific AbortSignal helpers should be checked against the supported Node.js LTS documentation.

## 12. Combine caller cancellation and timeout

~~~javascript
function createOperationSignal(
  callerSignal,
  timeoutMilliseconds,
) {
  return AbortSignal.any([
    callerSignal,
    AbortSignal.timeout(timeoutMilliseconds),
  ]);
}
~~~

The operation stops when the caller cancels or its own shorter deadline expires.

Cancellation reason matters. A client disconnect, application shutdown, explicit user action, and internal timeout may require different logs and responses.

## 13. Wait for an event with cancellation

~~~javascript
import { once } from "node:events";

const controller = new AbortController();

const [result] = await once(
  events,
  "report.completed",
  {
    signal: controller.signal,
  },
);
~~~

The listener is removed if the signal aborts. Lifecycle-aware helpers reduce manual cleanup mistakes.

## 14. Propagate cancellation through boundaries

~~~javascript
async function buildReport(
  input,
  { signal },
) {
  signal.throwIfAborted();

  const rows = await repository.loadRows({
    signal,
  });

  signal.throwIfAborted();

  return calculateReport(rows, {
    signal,
  });
}
~~~

Every layer must decide:

- Can it stop?
- What resources need cleanup?
- Can a side effect already have happened?
- Is the operation safe to retry?

## Real-world applications and edge cases

### Where this appears

- Request IDs flow through controllers, services, repositories, and logs.
- A client disconnect cancels a report or outbound request.
- Application shutdown aborts new background work.
- EventEmitter coordinates local lifecycle events.
- Tests wait for an event with a deadline.

### Edge cases to investigate

- Reconnection adds the same listener repeatedly.
- An async listener rejects after emit has returned.
- An error event has no listener and terminates the process.
- Async context from one request is stored globally and leaks into another.
- A detached timer continues after its request context should have ended.
- Cancellation arrives after a database commit but before the success response.
- A timeout rejects the caller while the underlying dependency continues working.
- Several signals abort with different reasons at nearly the same time.
- Cleanup removes the wrong listener because the function reference differs.
- A local event is assumed to reach another application instance.

Events announce; context identifies; cancellation limits lifetime. None of them automatically provide durability or transactional rollback.

## Pause and predict

- In what order do synchronous EventEmitter listeners run?
- What happens when error is emitted without a listener?
- Does emit await an async listener?
- Should authorization rely only on AsyncLocalStorage?
- Can AbortController undo a committed database write?
- Who must cooperate for cancellation to release resources?

## Guided practice

Extend the Study Tracker API:

1. Add AsyncLocalStorage request context.
2. Include request and user IDs in service and repository logs.
3. Create an application EventEmitter for local lifecycle events.
4. Register and clean up listeners explicitly.
5. Propagate AbortSignal into one slow report operation.
6. Abort it on timeout and simulated client disconnect.
7. Distinguish timeout, shutdown, and caller cancellation in logs.
8. Prove with tests that listener counts return to their original value.

Keep durable user-visible events in the shared messaging design from Lesson 17, not only EventEmitter.

## Common mistakes

- Treating EventEmitter as a durable message broker
- Adding listeners per request without removing them
- Blocking inside synchronous listeners
- Forgetting the special error event behavior
- Assuming emit awaits promises
- Storing authorization only in invisible async context
- Creating an AbortController without passing its signal anywhere
- Catching abort errors as unexpected server failures
- Timing out the caller without cancelling downstream work
- Assuming cancellation reverses side effects

## Independent challenge

Build a cancellable report coordinator:

- A request creates a report with a unique ID.
- EventEmitter announces local progress.
- AsyncLocalStorage attaches request, user, and report IDs to logs.
- A timeout or shutdown signal cancels computation.
- Listener cleanup occurs on success, failure, and cancellation.
- Durable job state remains in PostgreSQL.
- A completed side effect is never repeated blindly after cancellation.

Write tests for cancellation before start, during work, after completion, and while cleanup fails.

## Knowledge check

- What is EventEmitter's process boundary?
- Why can listener accumulation leak memory?
- What problem does AsyncLocalStorage solve?
- Why should critical identity still be passed explicitly?
- What does AbortSignal communicate?
- How do deadlines and cancellation differ?

## Explore with an AI agent

- Review my listener lifecycle and find paths that skip cleanup.
- Trace request context through one real asynchronous call chain.
- Give me cancellation races and ask what side effects may already exist.
- Compare EventEmitter, Redis Pub/Sub, and a durable queue for one event.
- Help me distinguish timeout errors from caller and shutdown cancellation.

## Official reading

- [Events API](https://nodejs.org/api/events.html)
- [Async context and AsyncLocalStorage](https://nodejs.org/api/async_context.html)
- [AbortController and AbortSignal globals](https://nodejs.org/api/globals.html#class-abortcontroller)
- [Promise-based timers](https://nodejs.org/api/timers.html#timers-promises-api)
- [EventTarget and EventEmitter guidance](https://nodejs.org/api/events.html#eventtarget-and-event-api)

## Definition of done

- Listener ownership and cleanup are explicit.
- Async listener rejections are handled.
- Request context survives expected asynchronous boundaries.
- Authorization does not depend solely on implicit context.
- Cancellation and deadlines propagate to cooperative operations.
- Tests cover lifecycle races and resource cleanup.

