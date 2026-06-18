# Lesson 17 - WebSockets and Real-Time Systems

> Reading time: about 40 minutes  
> Practice time: about 100 minutes

## Outcome

Explain the WebSocket upgrade and connection lifecycle, build an authenticated real-time server with the ws package, validate messages, detect dead or slow clients, and design broadcasting across multiple Node.js instances.

## When WebSockets are useful

HTTP request-response remains the simpler choice for most operations. WebSockets are useful when the server must send low-latency updates without waiting for a new client request.

Examples:

- Chat messages
- Collaborative editing
- Live dashboards
- Multiplayer state
- Job progress
- Presence and notifications

The Study Tracker API will use HTTP to create and change resources, then WebSockets to notify connected clients about those changes.

## 1. A WebSocket begins as HTTP

The client first requests an HTTP protocol upgrade. A successful server replies with status 101 Switching Protocols.

Conceptual handshake:

~~~text
GET /realtime HTTP/1.1
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: ...
Sec-WebSocket-Version: 13
~~~

After the upgrade, the connection no longer follows ordinary HTTP request-response behavior. Both sides can send frames until the connection closes.

Node.js exposes the server upgrade event. A WebSocket library performs protocol framing, validation, and connection management.

## 2. Install the ws package

~~~powershell
npm install ws
~~~

Node.js includes a WebSocket client in modern releases, but a WebSocket server still needs protocol implementation. This course uses the widely adopted ws package.

Check its current official documentation when installing because options may evolve.

## 3. Attach WebSockets to the existing HTTP server

src/realtime/create-websocket-server.js:

~~~javascript
import { WebSocketServer } from "ws";

export function createWebSocketServer({
  server,
  authenticateUpgrade,
  logger,
}) {
  const webSocketServer = new WebSocketServer({
    noServer: true,
    maxPayload: 64 * 1024,
  });

  server.on("upgrade", async (request, socket, head) => {
    function onSocketError(error) {
      logger.warn("websocket_socket_error", {
        errorMessage: error.message,
      });
    }

    socket.on("error", onSocketError);

    try {
      const url = new URL(request.url, "http://localhost");

      if (url.pathname !== "/realtime") {
        socket.destroy();
        return;
      }

      const identity = await authenticateUpgrade(request, url);

      socket.removeListener("error", onSocketError);

      webSocketServer.handleUpgrade(
        request,
        socket,
        head,
        (webSocket) => {
          webSocketServer.emit(
            "connection",
            webSocket,
            request,
            identity,
          );
        },
      );
    } catch (error) {
      logger.warn("websocket_upgrade_rejected", {
        errorMessage: error.message,
      });
      socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");
      socket.destroy();
    }
  });

  return webSocketServer;
}
~~~

noServer lets the Express application and WebSocket server share one HTTP server and port.

maxPayload bounds message size. A connection that can send unlimited data is a memory problem waiting to achieve its dreams.

## 4. Authentication during upgrade

Browser WebSocket clients cannot freely set an Authorization header like fetch can. Common approaches include:

- Existing secure session cookie
- Short-lived, single-use WebSocket ticket obtained through authenticated HTTP
- Carefully defined subprotocol token

Avoid long-lived bearer tokens in query strings. URLs can appear in logs and monitoring systems.

### One-time ticket flow

~~~text
1. Authenticated client POSTs /realtime/tickets.
2. Server returns a random ticket with a very short expiration.
3. Client connects to /realtime?ticket=...
4. Upgrade handler consumes the ticket exactly once.
5. Connection receives the associated user identity.
~~~

The ticket repository should store a hash rather than the raw ticket, following the same pattern as auth session tokens.

## 5. Validate Origin when browser credentials are automatic

Cookies are automatically sent by browsers. A malicious site may attempt to open a WebSocket using the victim's cookies.

When using cookie authentication:

- Validate the Origin header against an allowlist.
- Use secure cookie settings.
- Require TLS in production.
- Do not treat CORS middleware as WebSocket protection.

The HTTP upgrade happens outside normal Express route middleware unless you explicitly share logic.

## 6. Connection lifecycle

~~~javascript
webSocketServer.on(
  "connection",
  (webSocket, request, identity) => {
    webSocket.userId = identity.userId;

    logger.info("websocket_connected", {
      userId: identity.userId,
    });

    webSocket.on("message", (data, isBinary) => {
      handleMessage(webSocket, data, isBinary);
    });

    webSocket.on("error", (error) => {
      logger.warn("websocket_error", {
        userId: identity.userId,
        errorMessage: error.message,
      });
    });

    webSocket.on("close", (code, reason) => {
      logger.info("websocket_closed", {
        userId: identity.userId,
        code,
        reason: reason.toString(),
      });
    });
  },
);
~~~

Important events are connection, message, error, and close. Cleanup room membership, subscriptions, and counters when a connection closes.

## 7. Design a message envelope

WebSocket frames do not automatically provide REST-like routes. Define an application protocol.

~~~json
{
  "type": "course.subscribe",
  "id": "message-123",
  "payload": {
    "courseId": 42
  }
}
~~~

Fields:

- type selects behavior.
- id correlates an acknowledgment or error.
- payload contains validated data.

Possible response:

~~~json
{
  "type": "ack",
  "replyTo": "message-123",
  "payload": {
    "subscribed": true
  }
}
~~~

Version the protocol when backward-incompatible message shapes become necessary.

## 8. Parse and validate every message

~~~javascript
function parseMessage(data, isBinary) {
  if (isBinary) {
    throw new Error("Binary messages are not supported");
  }

  let message;

  try {
    message = JSON.parse(data.toString("utf8"));
  } catch {
    throw new Error("Message must contain valid JSON");
  }

  if (
    !message ||
    typeof message.type !== "string" ||
    typeof message.payload !== "object"
  ) {
    throw new Error("Message envelope is invalid");
  }

  return message;
}
~~~

Valid JSON is not valid application input. Apply type-specific schemas, authorization, size limits, and rate limits.

Do not let a user subscribe to a course before checking ownership or membership.

## 9. Keep HTTP authoritative

For the Study Tracker project:

~~~text
POST /sessions
  -> validates and commits to PostgreSQL
  -> publishes session.created
  -> WebSocket clients receive the committed update
~~~

This keeps database mutations inside the existing service and transaction design. The real-time channel communicates change rather than creating a second inconsistent command system.

## 10. Track connections by identity

~~~javascript
const connectionsByUser = new Map();

function addConnection(userId, webSocket) {
  let connections = connectionsByUser.get(userId);

  if (!connections) {
    connections = new Set();
    connectionsByUser.set(userId, connections);
  }

  connections.add(webSocket);
}

function removeConnection(userId, webSocket) {
  const connections = connectionsByUser.get(userId);
  if (!connections) return;

  connections.delete(webSocket);

  if (connections.size === 0) {
    connectionsByUser.delete(userId);
  }
}
~~~

One user may have several tabs or devices. Set a deliberate maximum active connection count per user.

This map represents only the current process. Lesson 19 explains multi-instance delivery.

## 11. Send safely

~~~javascript
import WebSocket from "ws";

function sendJson(webSocket, message) {
  if (webSocket.readyState !== WebSocket.OPEN) {
    return false;
  }

  if (webSocket.bufferedAmount > 512 * 1024) {
    return false;
  }

  webSocket.send(JSON.stringify(message));
  return true;
}
~~~

bufferedAmount reveals queued outgoing bytes. A slow client can otherwise accumulate memory.

Choose a slow-consumer policy:

- Drop nonessential updates.
- Send a resynchronization-required message.
- Close the connection.
- Store durable events for later replay.

The correct policy depends on whether events are replaceable, cumulative, or legally important.

## 12. Heartbeats detect broken connections

Networks can disappear without a clean close frame.

~~~javascript
function markAlive() {
  this.isAlive = true;
}

webSocketServer.on("connection", (webSocket) => {
  webSocket.isAlive = true;
  webSocket.on("pong", markAlive);
});

const heartbeatInterval = setInterval(() => {
  for (const webSocket of webSocketServer.clients) {
    if (webSocket.isAlive === false) {
      webSocket.terminate();
      continue;
    }

    webSocket.isAlive = false;
    webSocket.ping();
  }
}, 30_000);

heartbeatInterval.unref();
~~~

The server marks every connection suspect, sends ping, and expects pong before the next check. ws clients automatically answer protocol ping frames.

Clear the interval during shutdown.

## 13. Reconnection and missed events

Clients should reconnect with:

- Exponential backoff
- Random jitter
- Maximum delay
- Network-awareness where available

After reconnecting, the client may have missed events. Options:

- Refetch current state through HTTP.
- Resume from a durable event sequence.
- Request changes after a known version.

For the Study Tracker, refetching current course and session state is initially simpler and safer than pretending every transient event is durable.

## 14. Message and connection rate limits

Lesson 13 continues after upgrade:

- Limit upgrade attempts.
- Limit active connections per identity.
- Limit messages per time interval.
- Give expensive messages higher cost.
- Bound subscriptions.
- Close clients that repeatedly violate protocol.

An HTTP Express limiter does not count frames after status 101.

## 15. Multiple instances need shared delivery

Imagine User 42 connects to Instance A, but their HTTP request reaches Instance B:

~~~text
Instance B commits session.created.
Instance B checks its local connection map.
User 42 is connected only to Instance A.
No notification arrives.
~~~

Publish the event through shared infrastructure:

~~~text
Instance B -> shared pub/sub -> Instance A -> WebSocket
~~~

Redis Pub/Sub can distribute transient notifications, but its delivery is at-most-once. If a subscriber disconnects, the message can be lost. Use a durable stream or broker when replay and stronger delivery guarantees matter.

Sticky sessions keep one connection attached to one instance, but they do not make events from other instances visible.

## 16. Graceful shutdown

Shutdown sequence:

1. Stop accepting new upgrades.
2. Mark readiness false.
3. Notify clients that the server is restarting.
4. Close with an appropriate close code such as 1001.
5. Wait for a short deadline.
6. Terminate remaining sockets.
7. Clear heartbeat timers and shared subscriptions.

Clients should reconnect with backoff to avoid a restart storm.

## Real-world applications and edge cases

### Where this appears

- Chat and presence systems
- Collaborative documents
- Delivery, market, and operations dashboards
- Background-job progress
- Multiplayer and live classroom applications

### Edge cases to investigate

- A mobile client changes networks and leaves a half-open connection behind.
- Thousands of clients reconnect simultaneously after a deployment.
- A client receives duplicate or out-of-order events after reconnecting.
- Authentication expires while a connection remains open.
- A user loses permission to a subscribed resource during the connection.
- A slow client accumulates outbound messages faster than it can receive them.
- A proxy closes an idle connection before the application heartbeat interval.
- One process restarts while other instances continue publishing events.
- Redis Pub/Sub loses messages while the subscriber is disconnected.
- A user opens dozens of tabs and exceeds expected connection count.
- A schema change reaches new clients while older mobile clients remain connected.

Define connection lifetime, authorization refresh, ordering, deduplication, protocol versioning, replay, and overload behavior before promising “real time.”

## Pause and predict

- Does an Express CORS setting automatically protect the upgrade?
- Why can a long-lived token in a query string leak?
- What happens when an HTTP request reaches a different instance from the user's socket?
- Why is sticky routing insufficient for broadcast?
- Which updates may be safely dropped for a slow client?

## Guided practice

Add real-time notifications to the Study Tracker:

1. Create short-lived, single-use connection tickets.
2. Upgrade only /realtime.
3. Authenticate and validate Origin.
4. Track several connections per user.
5. Publish session.created and session.deleted after database commit.
6. Send events only to the owning user.
7. Add heartbeat cleanup.
8. Bound payload, message rate, connection count, and buffered output.
9. On reconnect, refetch current state through HTTP.

Write protocol examples before writing handlers.

## Common mistakes

- Using WebSockets when ordinary HTTP is sufficient
- Putting long-lived bearer tokens in URLs
- Forgetting Origin validation with cookie authentication
- Trusting JSON without message-specific validation
- Mutating data through a second uncontrolled code path
- Ignoring message size, rate, and subscription limits
- Assuming close always fires promptly after network loss
- Ignoring bufferedAmount and slow consumers
- Keeping connection maps only in one instance while scaling
- Assuming Redis Pub/Sub stores missed events
- Forgetting heartbeat and subscription cleanup

## Independent challenge

Add live report progress:

- POST /reports creates a background job over HTTP.
- The response returns a report ID.
- The owning user subscribes to that report.
- Workers publish queued, running, progress, complete, or failed.
- Reconnecting clients fetch current job state before resubscribing.
- Another user cannot subscribe.
- Duplicate and out-of-order progress messages have a defined policy.

Document which state is durable in PostgreSQL and which events are transient.

## Knowledge check

- How does the HTTP upgrade relate to WebSocket frames?
- Why use an application message envelope?
- What does a heartbeat detect?
- How does backpressure appear in ws?
- Why do multiple instances need shared messaging?
- What guarantee does Redis Pub/Sub not provide?

## Explore with an AI agent

- Review my WebSocket authentication for URL, cookie, and Origin risks.
- Generate invalid and adversarial messages for my protocol.
- Trace a session.created event across two application instances.
- Challenge my slow-consumer and reconnection policies.
- Compare transient pub/sub with durable streams for my report-progress feature.

## Official reading

- [WebSocket protocol](https://www.rfc-editor.org/rfc/rfc6455.html)
- [Node.js HTTP upgrade event](https://nodejs.org/api/http.html#event-upgrade)
- [ws project and examples](https://github.com/websockets/ws)
- [WHATWG WebSockets Standard](https://websockets.spec.whatwg.org/)
- [Redis Pub/Sub delivery semantics](https://redis.io/docs/latest/develop/interact/pubsub/)
- [WebSocket close codes](https://www.rfc-editor.org/rfc/rfc6455.html#section-7.4)

## Definition of done

- Upgrade authentication and Origin policy are explicit.
- Every message is size-limited, parsed, validated, authorized, and rate-limited.
- Heartbeats remove dead connections.
- Slow consumers cannot grow memory without bound.
- Reconnecting clients can recover missed state.
- The scaling design delivers events across application instances.
