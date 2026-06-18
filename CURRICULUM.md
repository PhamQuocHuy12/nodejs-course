# Curriculum

The chapters are ordered deliberately. Complete the green path in sequence rather than choosing technologies at random from the larger roadmap.

The dependency path is:

~~~text
Runtime foundations
  -> one secure API instance
  -> testing and observability
  -> streams and real-time connections
  -> workers and multiple instances
  -> load balancing
  -> advanced Node.js runtime behavior
  -> production operation
  -> capstone
~~~

Rate limiting follows authentication because useful quotas often need user identity. WebSockets follow streams and observability because they are long-lived duplex streams that require lifecycle evidence. Load balancing follows WebSockets and workers because scaling exposes every assumption that state lives in one process.

Every lesson includes a **Real-world applications and edge cases** section. Read it after the main theory, then choose one scenario to discuss with an AI agent before starting the independent challenge.

## Phase 1 - Foundations

1. [How Node.js Works](lessons/01-how-nodejs-works.md) - runtime, V8, browser differences
2. [Running Code and Using the REPL](lessons/02-running-code-and-repl.md) - process, arguments, standard streams
3. [Modules: CommonJS and ESM](lessons/03-modules.md) - code organization and package boundaries
4. [npm and Package Management](lessons/04-npm-and-packages.md) - packages, scripts, lockfiles, semantic versions

## Phase 2 - Core Node.js

5. [Async JavaScript and the Event Loop](lessons/05-async-and-event-loop.md) - callbacks, promises, timers, concurrency
6. [Errors and Debugging](lessons/06-errors-and-debugging.md) - failure design, stack traces, debugger
7. [Files, Paths, and Directories](lessons/07-files-and-paths.md) - portable paths and promise-based file operations
8. [Environment Variables and Configuration](lessons/08-environment-and-config.md) - validated settings and secrets

## Phase 3 - Building APIs

9. [HTTP Fundamentals](lessons/09-http-fundamentals.md) - requests, responses, routing, JSON
10. [Express and REST APIs](lessons/10-express-rest-api.md) - middleware and maintainable architecture
11. [PostgreSQL and TypeORM](lessons/11-postgresql-and-typeorm.md) - relational modeling and persistence
12. [Authentication and API Security](lessons/12-auth-and-security.md) - identity, ownership, defensive APIs
13. [Rate Limiting and API Protection](lessons/13-rate-limiting-and-api-protection.md) - quotas, algorithms, distributed limits, and overload control

## Phase 4 - Quality, Data Flow, and Real-Time Systems

14. [Automated Testing](lessons/14-testing.md) - unit, integration, and API tests
15. [Logging and Observability](lessons/15-logging-and-observability.md) - structured evidence from running systems
16. [Streams and Large Data](lessons/16-streams.md) - chunks, pipelines, and backpressure
17. [WebSockets and Real-Time Systems](lessons/17-websockets-and-real-time-systems.md) - connection lifecycle, authentication, heartbeats, and shared messaging

## Phase 5 - Scaling

18. [Processes, Workers, and Concurrency](lessons/18-processes-and-workers.md) - CPU work and isolation
19. [Load Balancing and Horizontal Scaling](lessons/19-load-balancing-and-horizontal-scaling.md) - proxies, multiple instances, health checks, and connection draining

## Phase 6 - Advanced Node.js Runtime

20. [Events, Async Context, and Cancellation](lessons/20-events-async-context-and-cancellation.md) - listener lifecycles, request context, deadlines, and cleanup
21. [Outbound HTTP and Resilience](lessons/21-outbound-http-and-resilience.md) - fetch, connection capacity, retries, idempotency, and dependency failure
22. [Memory, Performance, and Diagnostics](lessons/22-memory-performance-and-diagnostics.md) - V8 memory, leaks, profiling, event-loop health, and diagnostic evidence

## Phase 7 - Production and Capstone

23. [Production and Deployment](lessons/23-production-and-deployment.md) - startup, shutdown, migrations, operations
24. [Capstone Project](lessons/24-capstone.md) - independent design and demonstration

## Optional branches from the roadmap

After the core course, choose topics based on your goals:

- CLI applications: stdin, stdout, process.argv, Commander, Inquirer
- Server-rendered pages: EJS, Pug, or Marko
- Alternative frameworks: Fastify, NestJS, or Hono
- MongoDB and Mongoose
- Browser end-to-end testing with Playwright or Cypress
- Message brokers and durable event streaming
- Node.js permission model and supply-chain hardening
- HTTP/2 and binary protocols
