# Lesson 09 - HTTP Fundamentals

> Reading time: about 30 minutes  
> Practice time: about 60 minutes

## Outcome

Explain an HTTP exchange and build the first Study Tracker API using only Node.js built-in modules.

## Why begin without Express

Express makes routing and middleware convenient, but it does not replace HTTP. Building one small server with node:http lets you see what a framework later handles for you.

## 1. HTTP is a request-response protocol

A client sends a request containing:

- Method
- Target URL
- Headers
- Optional body

A server sends a response containing:

- Status code
- Headers
- Optional body

### Example exchange

~~~text
POST /sessions HTTP/1.1
Content-Type: application/json

{"course":"Node.js","minutes":45}
~~~

Possible response:

~~~text
HTTP/1.1 201 Created
Content-Type: application/json

{"id":1,"course":"Node.js","minutes":45}
~~~

## 2. Methods express intent

| Method | Typical intent |
|---|---|
| GET | Read a resource |
| POST | Create a new resource |
| PUT | Replace a resource |
| PATCH | Partially update a resource |
| DELETE | Remove a resource |

These are conventions with protocol semantics, not magic function names.

## 3. Status codes communicate outcomes

- 200 OK: successful read or update
- 201 Created: resource created
- 204 No Content: success with no response body
- 400 Bad Request: malformed request
- 401 Unauthorized: valid authentication is required
- 403 Forbidden: identity is known but not allowed
- 404 Not Found: resource or route is absent
- 409 Conflict: request conflicts with current state
- 422 Unprocessable Content: syntactically valid but semantically invalid
- 500 Internal Server Error: unexpected server failure

The body explains details; the status communicates the broad result.

## 4. Your first server

~~~javascript
import { createServer } from "node:http";

const server = createServer((request, response) => {
  response.writeHead(200, {
    "content-type": "application/json; charset=utf-8",
  });

  response.end(JSON.stringify({ message: "Hello from Node.js" }));
});

server.listen(3000, () => {
  console.log("Listening on http://localhost:3000");
});
~~~

createServer receives a function for every request. listen keeps the process alive while the server accepts connections.

## 5. Read the method and URL

~~~javascript
const url = new URL(request.url, "http://localhost");

console.log(request.method);
console.log(url.pathname);
console.log(url.searchParams.get("course"));
~~~

request.url is normally relative, so the URL constructor needs a base. The base is used for parsing; it is not automatically the public server address.

## 6. Send JSON consistently

~~~javascript
function sendJson(response, statusCode, data) {
  const body = JSON.stringify(data);

  response.writeHead(statusCode, {
    "content-type": "application/json; charset=utf-8",
    "content-length": Buffer.byteLength(body),
  });

  response.end(body);
}
~~~

The content type tells the client how to interpret bytes. Buffer.byteLength measures encoded bytes rather than JavaScript characters.

## 7. Request bodies are streams

An HTTP body can arrive in chunks. Do not assume the first chunk is the whole JSON document.

~~~javascript
async function readJson(request, maximumBytes = 100_000) {
  const chunks = [];
  let totalBytes = 0;

  for await (const chunk of request) {
    totalBytes += chunk.length;

    if (totalBytes > maximumBytes) {
      const error = new Error("Request body is too large");
      error.statusCode = 413;
      throw error;
    }

    chunks.push(chunk);
  }

  const text = Buffer.concat(chunks).toString("utf8");

  if (!text) {
    return null;
  }

  try {
    return JSON.parse(text);
  } catch (cause) {
    const error = new Error("Request body contains invalid JSON", {
      cause,
    });
    error.statusCode = 400;
    throw error;
  }
}
~~~

The size limit prevents an unbounded body from consuming memory.

## 8. First Study Tracker routes

~~~javascript
import { createServer } from "node:http";

const sessions = [];
let nextId = 1;

const server = createServer(async (request, response) => {
  try {
    const url = new URL(request.url, "http://localhost");

    if (request.method === "GET" && url.pathname === "/health") {
      return sendJson(response, 200, { status: "ok" });
    }

    if (request.method === "GET" && url.pathname === "/sessions") {
      return sendJson(response, 200, { data: sessions });
    }

    if (request.method === "POST" && url.pathname === "/sessions") {
      const input = await readJson(request);

      if (
        !input ||
        typeof input.course !== "string" ||
        !Number.isInteger(input.minutes) ||
        input.minutes <= 0
      ) {
        return sendJson(response, 422, {
          error: {
            code: "INVALID_SESSION",
            message: "course and positive integer minutes are required",
          },
        });
      }

      const session = {
        id: nextId++,
        course: input.course.trim(),
        minutes: input.minutes,
      };

      sessions.push(session);
      return sendJson(response, 201, { data: session });
    }

    return sendJson(response, 404, {
      error: {
        code: "ROUTE_NOT_FOUND",
        message: "Route not found",
      },
    });
  } catch (error) {
    const statusCode = error.statusCode ?? 500;

    if (statusCode === 500) {
      console.error(error);
    }

    return sendJson(response, statusCode, {
      error: {
        code: statusCode === 500 ? "INTERNAL_ERROR" : "BAD_REQUEST",
        message:
          statusCode === 500 ? "Unexpected server error" : error.message,
      },
    });
  }
});

server.listen(3000);
~~~

Move sendJson and readJson to focused modules rather than leaving everything in one file.

## Real-world applications and edge cases

### Where this appears

- Public and internal JSON APIs
- Webhook receivers
- Health and readiness endpoints
- File upload and download services
- Reverse proxies and API gateways

### Edge cases to investigate

- A client times out after the server commits a POST, then retries and creates a duplicate.
- The client disconnects while the server continues expensive work.
- A slow client sends headers or body bytes gradually and holds a connection open.
- Content-Length disagrees with the received body, or transfer encoding uses chunks.
- A response begins streaming before an error occurs, so the status and headers can no longer change.
- Header names are case-insensitive, but application code compares values incorrectly.
- A proxy rewrites host, protocol, and client IP information.
- Keep-alive connections consume resources even while idle.
- A webhook provider signs the raw body, but JSON middleware changes what is available for verification.
- User input becomes part of a Location header and creates an unsafe redirect.

HTTP correctness includes retry behavior, deadlines, disconnections, proxy boundaries, and byte-level limits—not only route matching.

## 9. Test manually

PowerShell can call the API:

~~~powershell
Invoke-RestMethod http://localhost:3000/health
Invoke-RestMethod http://localhost:3000/sessions
Invoke-RestMethod -Method Post -Uri http://localhost:3000/sessions -ContentType application/json -Body '{"course":"Node.js","minutes":45}'
~~~

## Pause and predict

- Which status should POST return after creation?
- Why is a request body read as multiple chunks?
- What happens if content-type says JSON but the body is malformed?
- Why should a 500 response hide the stack trace?

## Guided practice

Add:

- GET /sessions/:id
- DELETE /sessions/:id
- Filtering with GET /sessions?course=Node.js

Return 404 when an ID does not exist. Validate that the ID contains a positive integer. DELETE may return 204 with no body.

## Common mistakes

- Returning 200 for every outcome
- Forgetting the content type
- Parsing only one body chunk
- Accepting unlimited request bodies
- Trusting input merely because it is valid JSON
- Returning stack traces to clients
- Sending a body after a 204 response
- Mixing route matching, business rules, and storage in one giant function

## Independent challenge

Complete an in-memory HTTP API with create, list, read, update, and delete behavior for study sessions. Write an API table showing method, path, successful status, and failure statuses before writing code.

Then restart the server and explain why the data disappears. That is the bridge to database persistence.

## Knowledge check

- What are the four main parts of a request?
- How do status and response body differ?
- Why does URL parsing need a base?
- Why impose a body-size limit?
- What does 204 require?

## Explore with an AI agent

- Review my API table before reviewing my implementation.
- Send me malformed HTTP scenarios and ask which status my server should return.
- Explain how an IncomingMessage is also a readable stream.
- Compare PUT and PATCH using study-session examples.

## Official reading

- [Node.js HTTP API](https://nodejs.org/api/http.html)
- [Anatomy of an HTTP transaction](https://nodejs.org/en/learn/http/anatomy-of-an-http-transaction)
- [URL API](https://nodejs.org/api/url.html)
- [Buffer API](https://nodejs.org/api/buffer.html)

## Definition of done

- Your server handles success, validation, missing routes, and malformed JSON.
- Request bodies have a size limit.
- Every response uses an intentional status and content type.
- You can explain the entire request-response path.
