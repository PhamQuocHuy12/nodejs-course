# Lesson 22 - Memory, Performance, and Diagnostics

> Reading time: about 45 minutes  
> Practice time: about 110 minutes

## Outcome

Explain Node.js memory categories and garbage collection, detect event-loop and resource pressure, collect CPU and heap evidence safely, and diagnose leaks before attempting optimization.

## Performance work begins with a question

“Make it faster” is not a useful diagnosis.

Start with an observable symptom:

- Request latency increased.
- Memory grows after every job.
- CPU stays near its limit.
- The event loop responds late.
- The process runs out of file descriptors.
- One endpoint allocates unusually large Buffers.

Measure the symptom, reproduce it, identify the constrained resource, and change one factor at a time.

## 1. Main memory categories

~~~javascript
console.log(process.memoryUsage());
~~~

Important fields include:

- rss: memory resident in the process
- heapTotal: V8 heap currently allocated
- heapUsed: V8 heap currently used
- external: memory associated with objects outside the V8 heap
- arrayBuffers: memory for ArrayBuffer and SharedArrayBuffer, including many Buffers

Heap memory is only part of process memory. A service can have moderate heapUsed while large Buffers make rss grow.

## 2. The V8 heap

JavaScript objects normally live in the V8 heap.

~~~javascript
const sessions = [
  { id: 1, minutes: 25 },
  { id: 2, minutes: 40 },
];
~~~

V8 uses generational garbage collection because most objects become unreachable quickly. Objects that survive may move into longer-lived regions.

Garbage collection reclaims unreachable memory, not memory the application still references.

## 3. Reachability defines a memory leak

~~~javascript
const cache = new Map();

export function rememberSession(session) {
  cache.set(session.id, session);
}
~~~

If entries are never removed, the Map keeps every session reachable. Garbage collection is working correctly; application policy is missing.

A memory leak is often legitimate allocation with an unintended lifetime.

## 4. Bound caches

~~~javascript
const cache = new Map();
const maximumEntries = 1_000;

export function setCached(key, value) {
  if (cache.size >= maximumEntries) {
    const oldestKey = cache.keys().next().value;
    cache.delete(oldestKey);
  }

  cache.set(key, value);
}
~~~

This simplified eviction policy is not a complete production cache, but it makes the bound visible.

Real cache policy should define:

- Maximum entries or bytes
- Expiration
- Eviction strategy
- Stale-data behavior
- Per-tenant fairness
- Observability

## 5. Common retention sources

Objects remain reachable through:

- Maps and arrays
- Event listeners
- Timers
- Closures
- Pending promises
- Request or socket registries
- Module-level singletons
- Async context stores
- Queued jobs

Example listener leak:

~~~javascript
function handleRequest(request) {
  events.on("configuration.changed", () => {
    refreshForUser(request.user);
  });
}
~~~

The listener captures request and user data after the response finishes. It is also added again for every request.

## 6. Buffers use memory beyond ordinary heap objects

~~~javascript
const buffer = Buffer.alloc(
  100 * 1024 * 1024,
);
~~~

The Buffer object is represented in V8, but its bytes contribute to external and ArrayBuffer-related memory.

Watch rss, external, and arrayBuffers together. Heap-only dashboards can miss binary-data growth.

## 7. Garbage collection is not a manual cleanup strategy

Do not design correctness around forcing garbage collection. The runtime decides when collection is useful.

High allocation rates can create frequent garbage collection even without a leak:

~~~javascript
for (let index = 0; index < 1_000_000; index += 1) {
  const temporary = {
    index,
    values: new Array(100).fill(index),
  };

  consume(temporary);
}
~~~

Reducing unnecessary allocation can improve performance, but only measurement can show whether it matters.

## 8. Event-loop delay

~~~javascript
import {
  monitorEventLoopDelay,
} from "node:perf_hooks";

const delay = monitorEventLoopDelay({
  resolution: 20,
});

delay.enable();

setInterval(() => {
  console.log({
    meanMs: Number(delay.mean) / 1_000_000,
    p99Ms:
      Number(delay.percentile(99)) /
      1_000_000,
    maxMs: Number(delay.max) / 1_000_000,
  });

  delay.reset();
}, 10_000).unref();
~~~

High delay can indicate:

- CPU-heavy JavaScript
- Long synchronous operations
- Garbage-collection pauses
- Too many immediately scheduled callbacks
- Host CPU contention

It does not identify the cause by itself.

## 9. Event-loop utilization

~~~javascript
import { performance } from "node:perf_hooks";

let previous =
  performance.eventLoopUtilization();

setInterval(() => {
  const current =
    performance.eventLoopUtilization(
      previous,
    );

  previous =
    performance.eventLoopUtilization();

  console.log(current.utilization);
}, 10_000).unref();
~~~

Utilization estimates how busy the event loop was. Interpret it beside throughput, latency, CPU, and delay.

High utilization with high CPU suggests busy JavaScript. Low utilization with slow requests may point toward waiting on dependencies or queues.

## 10. Measure duration with a monotonic clock

~~~javascript
import { performance } from "node:perf_hooks";

const startedAt = performance.now();
await performWork();
const durationMs =
  performance.now() - startedAt;

console.log({ durationMs });
~~~

performance.now is monotonic for duration measurement. Date.now represents wall-clock time and can shift when the system clock is adjusted.

## 11. Observe active resources

~~~javascript
console.log(
  process.getActiveResourcesInfo(),
);
~~~

Active resources can include timers, servers, sockets, and other work keeping the event loop alive.

This helps when:

- Tests do not exit.
- Shutdown hangs.
- A connection was not closed.
- A repeating timer was forgotten.

The list gives categories, not complete ownership. Use it as a lead.

## 12. CPU profiles

Run an application with CPU profiling:

~~~powershell
node --cpu-prof src/server.js
~~~

Exercise the slow behavior, stop the process cleanly, and inspect the generated profile with compatible developer tools.

A CPU profile samples where execution time is spent. Look for:

- Hot functions
- Excessive serialization
- Regular-expression cost
- Repeated transformations
- Unexpected library work

Do not optimize a function merely because it appears in the profile. Consider how often it runs and whether improving it affects user-visible latency.

## 13. Inspector profiling

~~~powershell
node --inspect src/server.js
~~~

The inspector protocol supports:

- Breakpoints
- CPU profiles
- Heap snapshots
- Allocation investigation

Do not expose the inspector port publicly. It can provide powerful control over the process and access to sensitive application data.

## 14. Heap snapshots

Programmatic snapshot:

~~~javascript
import { writeHeapSnapshot } from "node:v8";

const filename = writeHeapSnapshot();
console.log(filename);
~~~

Heap snapshots show objects and reference paths. Compare snapshots around a repeatable workload to find objects that remain reachable.

Important safety warning:

- Snapshot generation pauses the main thread.
- It can require significant additional memory.
- A near-limit production process may be terminated.
- Snapshot files can contain sensitive data.

Prefer a controlled replica or local reproduction. If production capture is necessary, plan the operational risk.

## 15. A leak investigation

Use this sequence:

1. Prove memory grows over repeated equivalent work.
2. Allow time for garbage collection behavior to settle.
3. Separate heap growth from Buffer or native growth.
4. Reproduce with the smallest workload.
5. Capture evidence at controlled points.
6. Find unexpected retaining paths.
7. Fix lifecycle or bounds.
8. Repeat the same workload.

One high memory reading is not automatically a leak. Stable applications may keep allocated memory for reuse.

## 16. Diagnostic reports

~~~javascript
const filename =
  process.report.writeReport();

console.log("Report written:", filename);
~~~

A diagnostic report can include:

- JavaScript and native stack information
- Node.js and operating-system details
- Resource usage
- Loaded libraries
- Environment and runtime data

Reports are useful for fatal errors, signals, or manual incident capture.

Treat reports as sensitive artifacts. Review storage, access, and retention.

## 17. Resource exhaustion beyond memory

A Node.js process can exhaust:

- File descriptors
- TCP sockets
- Ephemeral ports
- Database pool slots
- Worker queue capacity
- Threads
- Disk space
- CPU quota

Symptoms may appear as unrelated network or file errors.

Track both current use and configured limits.

## 18. Container memory limits

The host may have plenty of memory while a container has a small limit. The operating system or platform can terminate the process without a normal JavaScript out-of-memory path.

Set application memory and concurrency expectations relative to the deployment limit. Leave room for:

- V8 heap
- Buffers
- Native libraries
- Thread stacks
- Code and shared libraries
- Snapshot or profiling overhead

Do not set V8 heap size equal to the entire container limit.

## 19. Load-test with a realistic mix

A useful performance test includes:

- Representative endpoints
- Realistic request sizes
- Authentication
- Database data volume
- Connection reuse
- WebSockets where relevant
- Warm-up
- Gradual load increase

Observe:

- Throughput
- Latency percentiles
- Errors
- Event-loop delay and utilization
- CPU
- Heap, rss, and external memory
- Database pool and query time
- Active handles and sockets

Average latency can hide a painful tail.

## 20. Optimization order

Prefer:

1. Correctness
2. Measurement
3. Algorithm and architecture improvement
4. Query and I/O improvement
5. Allocation reduction
6. Small code-level tuning

Removing one unnecessary database query usually matters more than replacing clear JavaScript with a clever micro-optimization.

## Real-world applications and edge cases

### Where this appears

- An API slowly grows memory over several days.
- CSV exports allocate large Buffers.
- A regular expression blocks one process under malicious input.
- Tests hang because a timer or socket remains active.
- A container restarts under traffic spikes.
- A report route uses every CPU core through too many workers.

### Edge cases to investigate

- heapUsed falls after garbage collection while rss remains high.
- External Buffer memory grows but heap snapshots appear normal.
- A cache is bounded by entry count, but one entry can be enormous.
- A heap snapshot doubles memory and kills the process being diagnosed.
- Profiling changes timing enough to hide a race or latency pattern.
- A diagnostic artifact contains secrets or user data.
- High event-loop delay comes from host CPU contention rather than one hot function.
- A process leaks file descriptors while memory remains stable.
- Autoscaling starts more instances and unexpectedly exhausts database connections.
- A memory limit is reached before V8 reaches its configured heap limit.
- One tenant creates disproportionate cache and queue pressure.

Use several signals. Memory, CPU, latency, event-loop delay, and resource counts describe different failure modes.

## Pause and predict

- Why can rss grow while heapUsed remains stable?
- Can garbage collection reclaim an object still referenced by a Map?
- What can high event-loop delay indicate?
- Why is performance.now preferable for duration?
- What risks come with heap snapshots?
- Which resource leaks do heap snapshots not necessarily reveal?

## Guided practice

Create a diagnostics lab:

1. Build a route that leaks entries into an unbounded Map.
2. Generate repeated traffic and record memoryUsage.
3. Add a large Buffer path and compare heapUsed with external and rss.
4. Monitor event-loop delay.
5. Add a synchronous CPU loop and observe delay and utilization.
6. Collect a CPU profile.
7. Capture heap snapshots in a controlled environment.
8. Find the retaining Map path.
9. Replace it with a bounded policy.
10. Verify memory stabilizes under the same workload.
11. Leave one timer open, inspect active resources, then clean it up.

Keep the workload and measurements identical before and after the fix.

## Common mistakes

- Calling every high memory reading a leak
- Monitoring only heapUsed
- Increasing the heap limit instead of finding unbounded retention
- Taking production heap snapshots without capacity planning
- Exposing the inspector port
- Optimizing without a repeatable workload
- Using average latency alone
- Ignoring file descriptors, sockets, and pool limits
- Running CPU-sized worker pools in every replica without host-level planning
- Treating profiling artifacts as nonsensitive
- Changing several performance factors at once

## Independent challenge

Diagnose an intentionally degraded Study Tracker:

- One request path leaks listeners.
- One cache is unbounded.
- One export allocates the complete result.
- One route blocks the event loop.
- One shutdown path leaves a timer and socket open.

Create a report containing:

- User-visible symptom
- Reproduction
- Measurements
- Evidence artifact
- Root cause
- Fix
- Before-and-after comparison
- Regression test or alert

Do not reveal the injected defects before collecting evidence.

## Knowledge check

- How do rss, heapUsed, and external differ?
- What makes an object eligible for garbage collection?
- What does event-loop delay measure?
- When is a CPU profile useful?
- What does a heap snapshot show?
- What information can a diagnostic report contain?

## Explore with an AI agent

- Interpret these memory metrics without assuming a leak.
- Guide me through comparing two heap snapshots using retaining paths.
- Review my load-test design for unrealistic traffic or missing signals.
- Separate CPU, event-loop, database, and connection-pool hypotheses from evidence.
- Challenge my proposed optimization and ask what measurement supports it.

## Official reading

- [Understanding and tuning memory](https://nodejs.org/en/learn/diagnostics/memory/understanding-and-tuning-memory)
- [Using heap snapshots](https://nodejs.org/en/learn/diagnostics/memory/using-heap-snapshot)
- [Process memory usage](https://nodejs.org/api/process.html#processmemoryusage)
- [Performance hooks](https://nodejs.org/api/perf_hooks.html)
- [V8 API](https://nodejs.org/api/v8.html)
- [Inspector API](https://nodejs.org/api/inspector.html)
- [Diagnostic report API](https://nodejs.org/api/report.html)
- [Command-line profiling options](https://nodejs.org/api/cli.html)

## Definition of done

- Memory dashboards distinguish heap, rss, and external memory.
- Event-loop delay and utilization are measured.
- CPU and heap evidence are collected safely.
- Caches, listeners, timers, queues, and pools have bounds or owners.
- Resource leaks are considered alongside memory leaks.
- Performance changes are verified against the same workload.

