# Lesson 19 - Load Balancing and Horizontal Scaling

> Reading time: about 40 minutes  
> Practice time: about 100 minutes

## Outcome

Place multiple Node.js instances behind a reverse proxy, choose a balancing strategy, configure trusted forwarding and WebSocket upgrades, share application state correctly, and drain traffic during deployment.

## Why scale horizontally

One process has finite CPU, memory, connections, and availability. Horizontal scaling runs several application instances and distributes traffic among them.

~~~text
Clients
   |
Load balancer
   |------ Instance A
   |------ Instance B
   |------ Instance C
             |
         PostgreSQL
         Shared limiter store
         Shared messaging
~~~

More instances can improve capacity and resilience, but only when state and failure behavior are designed for it.

## 1. Reverse proxy versus load balancer

A reverse proxy accepts traffic on behalf of upstream servers. It can:

- Terminate TLS
- Route by host or path
- Add or normalize headers
- Compress responses
- Enforce request limits
- Hide internal topology

A load balancer distributes traffic across several upstream targets. One component often performs both roles.

## 2. Layer 4 and Layer 7 balancing

Layer 4 balancing operates on network connections such as TCP. It knows addresses and ports but not ordinary HTTP paths.

Layer 7 balancing understands HTTP details such as:

- Host
- Path
- Method
- Headers
- Cookies

Layer 7 can route /api and /realtime differently, but it performs more protocol work.

## 3. Common balancing algorithms

### Round robin

Send each new request or connection to the next healthy instance.

~~~text
Request 1 -> A
Request 2 -> B
Request 3 -> C
Request 4 -> A
~~~

It works well when instances and request costs are similar.

### Least connections

Choose the instance with the fewest active connections. This can fit long-lived WebSockets better than plain round robin.

Connection count is only an approximation of load. One CPU-heavy request may cost more than many idle sockets.

### Weighted balancing

Give stronger or newly introduced instances different traffic shares.

~~~text
A weight 5
B weight 5
C weight 1
~~~

Weights are useful for uneven capacity or gradual rollout.

### Consistent hashing or sticky routing

Route a key such as a cookie to the same instance. This can reduce cache churn or preserve legacy in-memory sessions.

Stickiness is not a substitute for shared durable state. If that instance fails, its memory still disappears.

## 4. Make API instances stateless

Stateless does not mean the system has no state. It means any healthy API instance can handle the next request because authoritative state lives in shared services.

Move out of local process memory:

- Login sessions
- Rate-limit counters
- Job status
- Uploaded files needed by other instances
- WebSocket event distribution
- Scheduled-job ownership

Suitable shared systems:

- PostgreSQL for durable application state
- Redis-like storage for fast shared counters or cache
- Object storage for files
- A broker or stream for distributed messages and jobs

Local memory remains useful for disposable caches and current socket objects.

## 5. Health checks control routing

The balancer should send traffic only to ready instances.

~~~text
GET /health -> process is alive
GET /ready  -> instance can currently serve traffic
~~~

During startup:

~~~text
Process starts
Database connects
Application initializes
Readiness becomes true
Traffic begins
~~~

During shutdown:

~~~text
Readiness becomes false
Balancer removes the instance
Existing work drains
Resources close
Process exits
~~~

An overly sensitive liveness check can create restart loops during a dependency outage.

## 6. Forwarded request information

After TLS termination, Node.js may see:

- Proxy IP instead of client IP
- Internal HTTP instead of original HTTPS
- Internal host instead of public host

Proxies communicate original values with standardized Forwarded or common X-Forwarded headers.

In Express:

~~~javascript
app.set("trust proxy", 1);
~~~

This trusts one proxy hop. It is correct only when every request truly passes through exactly one trusted proxy.

Overtrusting allows clients to spoof forwarded values. Undertrusting causes incorrect IPs, protocol detection, secure-cookie behavior, logs, and rate-limit keys.

Document the topology and test request.ip, request.protocol, and request.hostname through the real proxy.

## 7. TLS termination

The public connection should use HTTPS and secure WebSockets.

Common topology:

~~~text
Client -- TLS --> Load balancer -- private connection --> Node.js
~~~

Some threat models require encryption from the balancer to the application as well.

The application needs the original protocol information so it can:

- Generate correct absolute URLs
- Require secure cookies
- Redirect safely
- Log accurately

Do not trust a client-supplied X-Forwarded-Proto header unless it arrived through the configured proxy chain.

## 8. NGINX example

This simplified example balances two local instances:

~~~nginx
map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

upstream study_tracker {
  least_conn;
  server 127.0.0.1:3001;
  server 127.0.0.1:3002;
}

server {
  listen 8080;

  location / {
    proxy_pass http://study_tracker;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
  }
}
~~~

This is a learning configuration, not a complete hardened production setup. Timeouts, TLS, header policy, access logs, body limits, and deployment-specific behavior still need configuration.

## 9. Identify instances during practice

Add an instance identifier:

~~~javascript
const instanceId =
  process.env.INSTANCE_ID ?? "local-unknown";

app.get("/instance", (request, response) => {
  response.json({ instanceId });
});
~~~

Run:

~~~powershell
$env:PORT=3001
$env:INSTANCE_ID="A"
node src/server.js
~~~

Run a second terminal with port 3002 and identifier B. Repeated requests through the balancer should show distribution.

INSTANCE_ID is diagnostic metadata, not user-facing application state.

## 10. WebSockets behind a load balancer

The balancer must:

- Support HTTP upgrade
- Keep upgraded connections open
- Use suitable idle and maximum-lifetime settings
- Drain connections during deployment
- Forward required headers

Once upgraded, one socket remains attached to its selected instance.

If an event originates elsewhere:

~~~text
HTTP request -> Instance B
Shared message -> Instance A
WebSocket -> connected client
~~~

Sticky sessions alone do not distribute events. Shared pub/sub or durable messaging remains necessary.

## 11. Rate limiting across the topology

Rate limiting can occur at several layers:

- Edge or CDN for large-scale network abuse
- Load balancer or gateway for broad HTTP quotas
- Application for authenticated user and business-operation limits

Application instances need shared counters. They also need a trustworthy client identity from the proxy.

Layered limits should be documented so clients are not rejected by a mysterious policy no one remembers owning.

## 12. Connection draining

A rolling deployment replaces instances while traffic continues.

Bad sequence:

~~~text
Kill instance
Active requests and WebSockets fail
Balancer discovers failure later
~~~

Better sequence:

~~~text
Mark not ready
Balancer stops new assignments
Wait for propagation
Drain HTTP work
Notify or close WebSockets
Close dependencies
Stop process
~~~

The balancer, orchestrator, and Node.js process need compatible timeouts. If the platform sends a hard kill after 10 seconds, a 30-second application drain cannot finish.

## 13. Retries can multiply traffic

A balancer may retry a failed upstream request. Retrying a safe GET can help. Retrying a POST after the first instance committed but disconnected can create duplicate data.

Use:

- Carefully limited retries
- Idempotency keys for retriable creation operations
- Timeouts with enough context
- Circuit breaking or outlier removal
- Observability that records retries

Never enable broad retries without understanding method semantics and failure timing.

## 14. Load balancer availability

A single self-hosted load balancer can become the new single point of failure.

Production options include:

- Managed regional load balancer
- Multiple balancer instances with failover
- DNS or anycast front doors
- Orchestrator-provided service routing

The goal is not merely adding a box named “load balancer.” The traffic entry path itself needs an availability design.

## 15. Observe the distributed system

Include in logs and metrics:

- Request ID
- Instance ID
- Upstream status
- Balancer response status
- Request duration
- Retry count
- Active connections
- Readiness changes
- WebSocket connection count
- Shared-store failures

One request may appear in proxy and application logs. Correlation IDs make those records useful together.

## 16. Capacity is measured, not guessed

Load-test a production-like environment:

1. Choose a realistic request mix.
2. Increase arrival rate gradually.
3. Watch latency percentiles, errors, CPU, memory, database load, event-loop delay, and connection count.
4. Find the first constrained resource.
5. Change one factor.
6. Repeat.

Adding application instances cannot fix a database already at its connection or CPU limit.

## Real-world applications and edge cases

### Where this appears

- A public API scales from one instance to many during traffic peaks.
- Rolling deployments replace instances without stopping the service.
- WebSocket gateways keep long-lived connections distributed across hosts.
- Regional services route users to nearby healthy capacity.
- Canary releases send a small percentage of traffic to a new version.

### Edge cases to investigate

- Readiness flaps rapidly and the balancer repeatedly adds and removes an instance.
- Round robin sends equal requests to instances doing unequal background work.
- Least-connections favors a newly started instance and causes a sudden traffic spike.
- A balancer retries a committed POST and creates a duplicate.
- Sticky routing concentrates many heavy users on one instance.
- An autoscaling event opens enough new database pools to exhaust PostgreSQL.
- Forwarded-header trust differs between direct internal traffic and public proxy traffic.
- A regional outage moves traffic faster than the surviving region can scale.
- DNS and connection reuse keep some clients on removed addresses longer than expected.
- Every WebSocket reconnects to healthy instances at once after one node fails.
- The load balancer is healthy, but its shared configuration or certificate is invalid.

Horizontal scaling moves bottlenecks and introduces coordination. Test failure transitions, not only balanced steady-state traffic.

## Pause and predict

- Does stateless mean PostgreSQL is unnecessary?
- Which algorithm might suit long-lived WebSockets?
- Why can trusting every X-Forwarded-For value break rate limiting?
- Can sticky sessions replace shared pub/sub?
- What happens if balancer drain time is longer than the platform grace period?
- Can retrying POST create duplicates?

## Guided practice

Build a two-instance local topology:

1. Run Study Tracker instances on ports 3001 and 3002.
2. Give each an INSTANCE_ID.
3. Put NGINX or another chosen proxy on port 8080.
4. Verify alternating or least-connection routing.
5. Confirm both instances use the same PostgreSQL database.
6. Replace the rate limiter memory store with shared storage.
7. Connect WebSocket clients through the proxy.
8. Publish an event on one instance and deliver it through the other.
9. Mark one instance unready and observe routing.
10. Drain it while an HTTP request and WebSocket are active.

Record every observation rather than assuming the topology worked.

## Common mistakes

- Keeping sessions or counters only in process memory
- Trusting arbitrary forwarded headers
- Treating sticky sessions as high availability
- Forgetting WebSocket upgrade and idle settings
- Sending new traffic to an instance during shutdown
- Configuring incompatible drain timeouts
- Retrying unsafe requests automatically
- Scaling Node.js instances while ignoring database limits
- Running only one load balancer without acknowledging the failure point
- Testing distribution without identifying the serving instance

## Independent challenge

Design a rolling deployment for three instances:

- At least two remain ready.
- WebSocket clients receive restart notice and reconnect with jitter.
- HTTP creation supports idempotency keys.
- Rate limits remain global.
- Events reach users connected anywhere.
- One instance fails during drain.

Draw the event and request paths, define timeouts, and list the evidence that proves no committed study session was duplicated or lost.

## Knowledge check

- How do reverse proxy and load-balancing responsibilities differ?
- What does stateless API design require?
- Which signals remove an instance from routing?
- Why is proxy trust a security setting?
- How do WebSocket events cross instances?
- What risk do automatic retries create?

## Explore with an AI agent

- Review my real proxy topology and derive the narrowest Express trust proxy setting.
- Simulate failure of one instance during an HTTP POST and ask me about idempotency.
- Compare round robin, least connections, and consistent hashing for my traffic mix.
- Trace a WebSocket event from Instance B to a client on Instance A.
- Analyze my load-test evidence and identify the first bottleneck without guessing.

## Official reading

- [NGINX HTTP load balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [NGINX WebSocket proxying](https://nginx.org/en/docs/http/websocket.html)
- [Express behind proxies](https://expressjs.com/en/guide/behind-proxies.html)
- [Node.js cluster API](https://nodejs.org/api/cluster.html)
- [Node.js HTTP server API](https://nodejs.org/api/http.html#class-httpserver)
- [Redis Pub/Sub](https://redis.io/docs/latest/develop/interact/pubsub/)

## Definition of done

- Several instances serve traffic through one entry point.
- Shared state remains correct regardless of the selected instance.
- Forwarded identity and protocol are trusted narrowly.
- HTTP and WebSocket traffic both route correctly.
- Unready instances receive no new traffic and drain cleanly.
- Retries and idempotency have deliberate policies.
