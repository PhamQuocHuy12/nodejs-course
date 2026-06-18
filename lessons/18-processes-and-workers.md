# Lesson 18 - Processes, Workers, and Concurrency

> Reading time: about 32 minutes  
> Practice time: about 75 minutes

## Outcome

Recognize CPU-bound work, move it away from the main event loop with worker threads, understand child processes, and choose a bounded concurrency design.

## 1. Async I/O does not solve CPU blocking

~~~javascript
function countPrimes(limit) {
  let count = 0;

  for (let candidate = 2; candidate <= limit; candidate += 1) {
    let prime = true;

    for (
      let divisor = 2;
      divisor * divisor <= candidate;
      divisor += 1
    ) {
      if (candidate % divisor === 0) {
        prime = false;
        break;
      }
    }

    if (prime) count += 1;
  }

  return count;
}
~~~

This function performs CPU work synchronously. If an HTTP handler calls it with a large limit, other handlers wait.

Making the function async changes only its return type. It does not move the loop to another thread.

## 2. Measure event-loop delay

~~~javascript
const expectedAt = Date.now() + 100;

setTimeout(() => {
  console.log("Delay:", Date.now() - expectedAt, "ms");
}, 100);

countPrimes(5_000_000);
~~~

If the calculation occupies the thread beyond the timer threshold, the timer callback runs late. This is direct evidence of blocking.

## 3. Worker threads run JavaScript in parallel

worker.js:

~~~javascript
import { parentPort, workerData } from "node:worker_threads";

function countPrimes(limit) {
  // Use the implementation from above.
}

const result = countPrimes(workerData.limit);
parentPort.postMessage({ result });
~~~

main.js:

~~~javascript
import { Worker } from "node:worker_threads";

export function countPrimesInWorker(limit) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(
      new URL("./worker.js", import.meta.url),
      {
        workerData: { limit },
      },
    );

    worker.once("message", ({ result }) => resolve(result));
    worker.once("error", reject);
    worker.once("exit", (code) => {
      if (code !== 0) {
        reject(new Error("Worker stopped with code " + code));
      }
    });
  });
}
~~~

The main event loop remains able to serve requests while the worker performs CPU work.

## 4. Message passing and data transfer

Worker messages use structured cloning. Large values can be expensive to copy.

ArrayBuffer instances can be transferred, moving ownership without copying:

~~~javascript
worker.postMessage(buffer, [buffer]);
~~~

After transfer, the sending side can no longer use that buffer normally. SharedArrayBuffer supports shared memory, but synchronization becomes your responsibility and complexity rises quickly.

## 5. Do not create one worker for every request

Starting workers has cost. Unlimited requests could create unlimited workers and exhaust memory or CPU.

A worker pool should have:

- Fixed or bounded worker count
- Bounded queue
- Per-task timeout
- Error propagation
- Cancellation policy
- Graceful shutdown

When the queue is full, reject or shed work rather than promising infinite capacity.

## 6. Child processes run separate programs

Use spawn for streaming output:

~~~javascript
import { spawn } from "node:child_process";

const child = spawn("node", ["--version"], {
  stdio: ["ignore", "pipe", "pipe"],
});

child.stdout.on("data", (chunk) => {
  process.stdout.write(chunk);
});
~~~

Use exec for small, bounded shell output. Never build a shell command from untrusted input. Prefer spawn with an executable and argument array.

Child processes have separate memory and can run non-Node programs.

## 7. Cluster and multiple server processes

cluster can distribute network connections among Node.js processes. Modern deployment platforms often run multiple independent application instances instead.

With multiple processes:

- In-memory sessions are not shared.
- In-memory queues are not shared.
- Local file writes can conflict.
- Scheduled jobs can run more than once.

State that must be shared belongs in PostgreSQL, a cache, or another coordinated service.

## 8. Concurrency versus parallelism

Concurrency means tasks make progress over overlapping time. Async I/O provides concurrency even if JavaScript callbacks take turns on one thread.

Parallelism means tasks literally execute at the same time. Worker threads and multiple processes can provide parallel CPU execution.

## 9. Decision guide

| Work | First choice |
|---|---|
| Network and file I/O | Asynchronous APIs |
| CPU-heavy JavaScript | Worker thread or separate service |
| Run an external program | Child process |
| Use multiple CPU cores for servers | Multiple instances or processes |
| Tiny calculation | Main thread |

Moving work has overhead. Measure before adding complexity.

## Real-world applications and edge cases

### Where this appears

- Image resizing and document conversion
- Large report calculation
- Data compression and parsing
- Machine-learning inference
- Running trusted external command-line tools
- Isolating unreliable third-party processing

### Edge cases to investigate

- The cost of copying a large payload to a worker exceeds the calculation benefit.
- Every API process creates a full CPU-sized pool and collectively oversubscribes the host.
- One task never returns and permanently occupies a worker.
- A worker crashes after completing an external side effect but before reporting success.
- Cancellation reaches the queue but not the code currently running inside a worker.
- Short tasks wait behind one enormous task, creating unfair latency.
- A child process writes unlimited output and fills a parent buffer.
- User input becomes a shell command and enables command injection.
- Worker memory grows independently from the main thread and escapes ordinary heap monitoring.
- Process shutdown leaves child processes or work queues orphaned.

Jobs need IDs, deadlines, bounded queues, cancellation, idempotency, progress, result persistence, and a policy for tasks interrupted between side effect and acknowledgment.

## Pause and predict

- Does marking countPrimes async prevent blocking?
- Why is one worker per request dangerous?
- Which state disappears when several processes replace one process?
- Why prefer spawn arguments over a shell command string?

## Guided practice

1. Expose a deliberately CPU-heavy report route.
2. Send a health request while the report runs.
3. Record health latency.
4. Move report generation to a worker.
5. Repeat the measurement.
6. Add a two-worker pool and a bounded queue.
7. Shut the pool down when the server receives a termination signal.

The goal is not merely a faster report. It is predictable responsiveness under load.

## Common mistakes

- Believing async functions move CPU work
- Creating unlimited workers
- Ignoring worker error and exit events
- Copying enormous messages repeatedly
- Passing user input through a shell
- Keeping authoritative state only in process memory
- Using cluster before understanding deployment instances
- Adding parallelism without measurement

## Independent challenge

Create background report jobs:

- POST /reports returns a job ID.
- A bounded worker pool calculates reports.
- GET /reports/:id returns queued, running, complete, or failed.
- Job state survives API process restarts.
- Duplicate work has a deliberate policy.

Explain what belongs in PostgreSQL and what can remain inside the worker pool.

## Knowledge check

- How do concurrency and parallelism differ?
- What work belongs in worker threads?
- How do worker messages move data?
- Why bound the queue?
- When is a child process appropriate?

## Explore with an AI agent

- Analyze this function and estimate whether it is I/O-bound or CPU-bound.
- Review my worker pool for unbounded queues, double resolution, and shutdown races.
- Explain data copying versus ArrayBuffer transfer with a concrete report payload.
- Compare worker threads, child processes, and separate services for my report feature.

## Official reading

- [Worker threads API](https://nodejs.org/api/worker_threads.html)
- [Child process API](https://nodejs.org/api/child_process.html)
- [Cluster API](https://nodejs.org/api/cluster.html)
- [Do not block the event loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)

## Definition of done

- CPU-heavy work no longer blocks normal API requests.
- Worker count and queue size are bounded.
- Worker errors, timeouts, and shutdown are handled.
- Shared state does not depend on one process's memory.
