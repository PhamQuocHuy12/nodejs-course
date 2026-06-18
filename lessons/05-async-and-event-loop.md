# Lesson 05 - Async JavaScript and the Event Loop

> Reading time: about 28 minutes  
> Practice time: about 50 minutes

## Outcome

Explain how Node.js schedules asynchronous work, write promise-based code with async and await, and choose deliberately between sequential and concurrent operations.

## Why this matters

Node.js is effective when it can keep coordinating work instead of waiting. That advantage disappears when application code blocks the event loop or handles promises carelessly.

## 1. Synchronous code uses the call stack

JavaScript evaluates ordinary statements in order:

~~~javascript
console.log("A");

function study() {
  console.log("B");
}

study();
console.log("C");
~~~

Output:

~~~text
A
B
C
~~~

Calling study adds a function frame to the call stack. The frame is removed when the function returns.

## 2. A callback represents later work

~~~javascript
console.log("Start");

setTimeout(() => {
  console.log("Timer finished");
}, 100);

console.log("End");
~~~

Output:

~~~text
Start
End
Timer finished
~~~

setTimeout registers work and returns. The callback becomes eligible after the delay, but it runs only when JavaScript can receive it.

The delay is a minimum threshold, not a precise appointment.

## 3. The event loop coordinates callbacks

At a high level:

1. JavaScript runs current synchronous code.
2. Node.js and the operating system make progress on timers and I/O.
3. Completed work queues callbacks.
4. The event loop runs eligible callbacks when the call stack is free.

This simplified model is enough for application code. The official event-loop guide explains the individual phases.

## 4. Promises represent future outcomes

A promise is pending, fulfilled, or rejected.

~~~javascript
import { readFile } from "node:fs/promises";

const promise = readFile("notes.txt", "utf8");

promise
  .then((text) => {
    console.log(text);
  })
  .catch((error) => {
    console.error(error.message);
  });
~~~

then handles fulfillment. catch handles rejection. Returning a value from then passes it to the next step; throwing rejects the chain.

## 5. Async and await improve sequential readability

~~~javascript
import { readFile } from "node:fs/promises";

async function showNotes() {
  try {
    const text = await readFile("notes.txt", "utf8");
    console.log(text);
  } catch (error) {
    console.error("Could not read notes:", error.message);
  }
}

await showNotes();
~~~

An async function always returns a promise. Await pauses that function until the promise settles. It does not block the entire Node.js process.

## 6. Promise callbacks and timers have different queues

Pause before running:

~~~javascript
console.log("A");

setTimeout(() => console.log("timer"), 0);
Promise.resolve().then(() => console.log("promise"));

console.log("B");
~~~

Expected output:

~~~text
A
B
promise
timer
~~~

After synchronous code finishes, promise microtasks are processed before the timer callback. This is useful for understanding order, but application design should not depend on clever queue tricks.

## 7. Sequential versus concurrent waiting

Sequential:

~~~javascript
const first = await readCourse("node");
const second = await readCourse("database");
~~~

The second operation starts after the first finishes.

Concurrent:

~~~javascript
const [first, second] = await Promise.all([
  readCourse("node"),
  readCourse("database"),
]);
~~~

Both operations start before either is awaited. Promise.all rejects when one input rejects.

Use concurrency only when the operations are independent. If the second needs data from the first, sequential code is correct.

## 8. Blocking the event loop

~~~javascript
const startedAt = Date.now();

while (Date.now() - startedAt < 5000) {
  // Busy work blocks JavaScript for five seconds.
}

console.log("Finished");
~~~

During the loop, timers, request handlers, and promise callbacks cannot run on the main thread. Async syntax cannot rescue a synchronous CPU loop.

## Real-world applications and edge cases

### Where this appears

- An API loads a user, permissions, and independent dashboard widgets.
- A service calls several downstream APIs with deadlines.
- A worker consumes jobs concurrently but must not overwhelm PostgreSQL.
- A checkout performs ordered steps because later work depends on earlier results.

### Edge cases to investigate

- Promise.all fails fast after one rejection, but the other operations continue and may still create side effects.
- A loop starts ten thousand promises at once and exhausts sockets or database connections.
- A promise is created without await, return, or catch, so its failure becomes disconnected from the request.
- A caller disconnects, but expensive database or network work continues because cancellation is not propagated.
- A timeout rejects the caller while the underlying operation continues and later mutates state.
- Repeated process.nextTick or microtask scheduling starves timers and I/O.
- Retrying a rejected creation request duplicates work because the operation is not idempotent.
- Cleanup in finally also fails, hiding or competing with the original error.

Concurrency should be bounded, cancellation should travel through the call chain, and retries must account for side effects that may already have happened.

## Pause and predict

Without running code, predict the order:

~~~javascript
console.log("start");

setTimeout(() => console.log("timer"), 0);

await Promise.resolve();
console.log("after await");

console.log("end");
~~~

Then explain which lines run as initial synchronous work, which continuation comes from a promise microtask, and which callback waits for a timer phase.

## Guided practice: delay experiment

~~~javascript
function delay(milliseconds, value) {
  return new Promise((resolve) => {
    setTimeout(() => resolve(value), milliseconds);
  });
}

async function sequential() {
  const start = Date.now();
  await delay(300);
  await delay(300);
  await delay(300);
  return Date.now() - start;
}

async function concurrent() {
  const start = Date.now();
  await Promise.all([delay(300), delay(300), delay(300)]);
  return Date.now() - start;
}

console.log("Sequential:", await sequential());
console.log("Concurrent:", await concurrent());
~~~

Exact times vary. Explain why the first is near 900 milliseconds and the second is near 300.

## Common mistakes

- Forgetting to await or return a promise
- Using await inside a loop when operations are independent
- Using Promise.all when order or dependency requires sequence
- Starting unlimited concurrent work
- Believing a zero-delay timer is immediate
- Wrapping synchronous CPU work in an async function and expecting it not to block
- Catching an error and silently discarding it

## Independent challenge

Create a study-plan loader:

1. Simulate loading three independent courses with different delays.
2. Load them concurrently.
3. Print results in the original input order.
4. Make one load reject and handle the failure.
5. Add a version using Promise.allSettled that reports every outcome.

Write down which behavior is correct for an all-or-nothing screen and which is correct for a partial-results screen.

## Knowledge check

- What does the call stack contain?
- Why is a timer delay only a minimum?
- What state can a promise have?
- What exactly does await pause?
- When is Promise.all appropriate?
- Why does a CPU loop block unrelated requests?

## Explore with an AI agent

- Trace one fs promise from the call stack through Node.js and back to my await expression.
- Give me six async examples and ask whether they run sequentially or concurrently.
- Review my code for floating promises and accidental serial waits.
- Explain microtasks, timers, setImmediate, and process.nextTick without giving me rules to memorize blindly.

## Official reading

- [The Node.js event loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Do not block the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [Discover promises in Node.js](https://nodejs.org/en/learn/asynchronous-work/discover-promises-in-nodejs)
- [Timers API](https://nodejs.org/api/timers.html)

## Definition of done

- You can predict the basic timer and promise example.
- You can convert a promise chain to async and await.
- You choose sequence or concurrency based on dependency.
- You can identify code that blocks the event loop.
