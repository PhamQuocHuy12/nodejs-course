# Lesson 23 - Production and Deployment

> Reading time: about 32 minutes  
> Practice time: about 75 minutes

## Outcome

Create a predictable startup and shutdown lifecycle, configure HTTP limits and health checks, plan migrations and rollback, and document a repeatable deployment.

## Production is an operating environment

An application is not production-ready merely because it works on localhost. Production introduces:

- Untrusted traffic
- Process restarts
- Partial dependency failures
- Concurrent deployments
- Secret management
- Monitoring and incident response
- Data backup and restoration

Reliability begins by defining how the process enters and leaves service.

## 1. Pin a supported runtime

Declare the supported Node.js range in package.json:

~~~json
{
  "engines": {
    "node": ">=22"
  }
}
~~~

The number is an example, not a timeless recommendation. Select an actively supported LTS line and update it deliberately. Match local development, continuous integration, and deployment.

Commit package-lock.json and install applications with:

~~~powershell
npm ci
~~~

This gives deployment the locked dependency graph instead of resolving new versions unexpectedly.

## 2. Startup should be ordered

A useful startup sequence:

1. Load and validate configuration.
2. Initialize logging.
3. Connect to required dependencies, including shared limiter and messaging services.
4. Confirm schema compatibility.
5. Create the application.
6. Start listening.
7. Mark the instance ready.

Do not accept traffic before required dependencies are ready.

~~~javascript
import { AppDataSource } from "./data-source.js";
import { createApp } from "./app.js";
import { config } from "./config.js";

async function start() {
  await AppDataSource.initialize();

  const app = createApp({
    dataSource: AppDataSource,
  });

  const server = app.listen(config.port, () => {
    console.log("Server is ready on port", config.port);
  });

  return server;
}

start().catch((error) => {
  console.error("Startup failed", error);
  process.exitCode = 1;
});
~~~

A startup error should leave the process unhealthy and nonready rather than running half-initialized.

## 3. Configure server limits and timeouts

~~~javascript
const server = app.listen(config.port);

server.requestTimeout = 30_000;
server.headersTimeout = 15_000;
server.keepAliveTimeout = 5_000;
~~~

Exact values depend on workload and infrastructure. Limits protect finite resources, but a timeout shorter than legitimate work creates failures. Coordinate values with any reverse proxy.

Also limit:

- Request body size
- Uploaded file size
- Database query time
- Worker queue size
- Active WebSocket connections per user
- WebSocket message and payload rates
- Outbound request duration
- Login attempts

## 4. Handle termination signals

Deployment platforms commonly send SIGTERM before stopping a process. Local terminals commonly produce SIGINT.

~~~javascript
export function installShutdown({
  server,
  dataSource,
  logger,
}) {
  let shuttingDown = false;

  async function shutdown(signal) {
    if (shuttingDown) return;
    shuttingDown = true;

    logger.info("shutdown_started", { signal });

    const forceTimer = setTimeout(() => {
      logger.error("shutdown_timed_out");
      process.exit(1);
    }, 10_000);

    forceTimer.unref();

    try {
      await new Promise((resolve, reject) => {
        server.close((error) => {
          if (error) reject(error);
          else resolve();
        });
      });

      if (dataSource.isInitialized) {
        await dataSource.destroy();
      }

      logger.info("shutdown_completed");
    } catch (error) {
      logger.error("shutdown_failed", {
        errorMessage: error.message,
        stack: error.stack,
      });
      process.exitCode = 1;
    } finally {
      clearTimeout(forceTimer);
    }
  }

  process.once("SIGTERM", () => shutdown("SIGTERM"));
  process.once("SIGINT", () => shutdown("SIGINT"));
}
~~~

server.close stops accepting new connections and waits for existing work according to server behavior. The deployment platform must allow enough grace time.

## 5. Readiness during shutdown

As soon as shutdown begins, readiness should fail so the load balancer stops routing new requests. Then the server drains existing requests.

Lesson 19 defines the other side of this contract: the balancer must observe readiness, stop assigning new work, preserve WebSocket upgrade behavior, and allow enough drain time.

The sequence is:

~~~text
SIGTERM
  -> readiness becomes false
  -> load balancer stops new traffic
  -> server stops accepting connections
  -> WebSocket clients receive restart notice
  -> shared limiter and message subscriptions stop
  -> current requests finish
  -> sockets, workers, database, and shared clients close
  -> process exits
~~~

If the platform immediately kills the process, graceful application code cannot help. Configure platform grace periods.

## 6. Migrations are a deployment step

Avoid having every application instance race to apply migrations on startup.

A safer sequence:

1. Back up or confirm recovery capability.
2. Run migration once as a release step.
3. Verify migration success.
4. Deploy compatible application instances.
5. Monitor.

Backward-compatible expansion:

1. Add a nullable column.
2. Deploy code that can handle old and new rows.
3. Backfill data.
4. Add stricter constraints later.

This reduces risk during rolling deployments where old and new code overlap.

## 7. Rollback includes data compatibility

Rolling back code does not automatically reverse a destructive schema change.

Before deployment, answer:

- Can the old code run against the new schema?
- Is migration reversal safe?
- Was data transformed or deleted?
- How long does restoration take?
- Who decides to roll back?

Backups are useful only if restoration is tested.

## 8. Reverse proxies affect request information

A hosting platform may terminate TLS and forward requests to Node.js. Configure Express proxy trust only for known infrastructure.

Incorrect trust configuration can let clients spoof IP-related headers and weaken rate limits or secure-cookie decisions.

## 9. Production error policy

Expected request failures:

- Return controlled response
- Log at an appropriate level
- Keep serving

Fatal startup or corrupted process state:

- Log technical context
- Stop accepting traffic
- Clean up within a deadline
- Exit nonzero
- Let the supervisor restart or replace the process

Do not keep a broken process alive merely to avoid a restart.

## 10. Deployment documentation

DEPLOYMENT.md should include:

- Runtime and system requirements
- Build and start commands
- Required environment variables
- Database creation and migration
- Health and readiness paths
- Port and proxy expectations
- Load-balancer routing and drain settings
- WebSocket upgrade and idle-timeout settings
- Shared rate-limit and pub/sub dependencies
- Log destination
- Shutdown grace period
- Rollback procedure
- Backup restoration
- Ownership and incident contacts

Another developer should be able to deploy without reading your mind, which is famously unavailable as an npm package.

## Real-world applications and edge cases

### Where this appears

- Rolling and blue-green deployments
- Database migrations during continuous delivery
- Secret and certificate rotation
- Automated scaling and health-based replacement
- Incident rollback and backup restoration

### Edge cases to investigate

- SIGTERM arrives while a long request, transaction, or CSV stream is active.
- The platform's hard-kill deadline is shorter than the application's drain deadline.
- A migration succeeds on the schema but the new application fails to start.
- Old and new instances disagree about a message or database format.
- A secret rotates while existing database or messaging connections use old credentials.
- The process is killed by the operating system for memory use and receives no graceful signal.
- Disk fills with logs or temporary files.
- DNS continues returning an unavailable dependency address from cache.
- Readiness remains true while one critical dependency is saturated rather than disconnected.
- A rollback deploys old code that cannot read data written by the new version.
- Backup restoration succeeds technically but violates the required recovery time or loses too much recent data.

Define recovery time objective, recovery point objective, rollout compatibility, hard resource limits, and the evidence required to call a deployment healthy.

## Pause and predict

- Should the server listen before database initialization?
- What should readiness report after SIGTERM?
- Why can code rollback fail after a destructive migration?
- Why must proxy trust be narrow?

## Guided practice

Add to the Study Tracker API:

- Explicit startup function
- Startup-failure logging
- Request and header timeouts
- Mutable readiness state
- SIGTERM and SIGINT shutdown
- HTTP, WebSocket, database, worker-pool, limiter-store, and messaging cleanup
- Force-shutdown deadline
- DEPLOYMENT.md for one chosen platform

Test graceful shutdown while one slow request is active.

## Common mistakes

- Listening before dependencies are ready
- Calling process.exit during normal cleanup
- Leaving readiness true during shutdown
- Running migrations concurrently in every instance
- Assuming code rollback reverses data changes
- Trusting every proxy
- Using unbounded timeouts and queues
- Having backups without testing restoration
- Deploying a different Node.js line than was tested

## Independent challenge

Design a zero-downtime release plan for adding a required timezone column to study sessions with existing rows.

Include:

- Expansion migration
- Compatible application release
- Backfill
- Validation
- Final constraint
- Rollback point at each stage

Then simulate one failed stage and write the operator decision.

## Knowledge check

- What is the correct startup order?
- What does graceful shutdown protect?
- Why separate liveness and readiness?
- How do rolling deployments affect schema compatibility?
- What makes a backup trustworthy?

## Explore with an AI agent

- Review my startup and shutdown code for races and resources I forgot.
- Challenge my migration plan with a rolling-deployment scenario.
- Use the official documentation for my selected platform to review DEPLOYMENT.md.
- Give me an incident where rollback is unsafe and ask me to choose a recovery path.

## Official reading

- [Node.js process signals](https://nodejs.org/api/process.html#signal-events)
- [HTTP server timeouts](https://nodejs.org/api/http.html#class-httpserver)
- [Node.js production security](https://nodejs.org/en/learn/getting-started/security-best-practices)
- [npm ci](https://docs.npmjs.com/cli/latest/commands/npm-ci)
- [TypeORM migrations](https://typeorm.io/docs/advanced-topics/migrations/)

## Definition of done

- Startup refuses traffic until required dependencies are ready.
- Shutdown drains traffic and closes resources within a deadline.
- Deployment uses a locked dependency tree and supported Node.js line.
- Migration, rollback, and restoration steps are documented.
