# Lesson 10 - Express and REST APIs

> Reading time: about 30 minutes  
> Practice time: about 75 minutes

## Outcome

Rebuild the Study Tracker API with Express, understand middleware, and separate HTTP translation from business rules and storage.

## 1. What Express adds

Express sits on Node.js HTTP and supplies:

- Route matching
- Middleware composition
- Body parsing
- Response helpers
- Error-handling conventions

Install it:

~~~powershell
npm install express
~~~

Express reduces plumbing. It does not choose your resource design, validation rules, permissions, or architecture.

## 2. Minimal Express application

~~~javascript
import express from "express";

const app = express();

app.get("/health", (request, response) => {
  response.status(200).json({ status: "ok" });
});

app.listen(3000, () => {
  console.log("Listening on http://localhost:3000");
});
~~~

The route registers a handler for GET /health.

## 3. Middleware forms a pipeline

~~~javascript
app.use((request, response, next) => {
  const startedAt = Date.now();

  response.on("finish", () => {
    console.log(
      request.method,
      request.path,
      response.statusCode,
      Date.now() - startedAt,
    );
  });

  next();
});
~~~

A regular middleware receives request, response, and next. It can:

- End the response
- Change request or response state
- Call next to continue
- Pass an error to next

Forgetting next leaves the request waiting unless the middleware sends a response.

## 4. Parse JSON with a limit

~~~javascript
app.use(express.json({ limit: "100kb" }));
~~~

After this middleware, parsed JSON is available as request.body. Parsing does not validate your business fields.

## 5. Route parameters and query parameters

~~~javascript
app.get("/sessions/:id", (request, response) => {
  console.log(request.params.id);
  console.log(request.query.includeCourse);
});
~~~

Path parameters identify a resource. Query parameters usually filter, sort, paginate, or modify representation.

## 6. Separate responsibilities

Use a modest structure:

~~~text
src/
  app.js
  server.js
  routes/
    session-routes.js
  controllers/
    session-controller.js
  services/
    session-service.js
  repositories/
    memory-session-repository.js
  middleware/
    error-handler.js
  errors/
    app-error.js
~~~

Responsibilities:

- Route: method and path
- Controller: translate HTTP input and output
- Service: enforce application rules
- Repository: store and retrieve data
- Error middleware: translate failures consistently

Do not create layers that merely forward every argument forever. A boundary should own a real responsibility.

## 7. A service contains business rules

~~~javascript
export function createSessionService(repository) {
  return {
    async create(input) {
      if (typeof input.course !== "string" || !input.course.trim()) {
        const error = new Error("Course is required");
        error.statusCode = 422;
        error.code = "INVALID_COURSE";
        throw error;
      }

      if (!Number.isInteger(input.minutes) || input.minutes <= 0) {
        const error = new Error("Minutes must be a positive integer");
        error.statusCode = 422;
        error.code = "INVALID_MINUTES";
        throw error;
      }

      return repository.create({
        course: input.course.trim(),
        minutes: input.minutes,
      });
    },
  };
}
~~~

The service does not know about request or response. A CLI or test can use the same rule.

## 8. A controller translates HTTP

~~~javascript
export function createSessionController(service) {
  return {
    async create(request, response, next) {
      try {
        const session = await service.create(request.body);
        response.status(201).json({ data: session });
      } catch (error) {
        next(error);
      }
    },
  };
}
~~~

The controller reads HTTP input, calls application behavior, and selects the successful HTTP response.

## 9. Central error middleware

Error middleware has four parameters:

~~~javascript
export function errorHandler(error, request, response, next) {
  const statusCode = error.statusCode ?? 500;

  if (statusCode >= 500) {
    console.error(error);
  }

  response.status(statusCode).json({
    error: {
      code: error.code ?? "INTERNAL_ERROR",
      message:
        statusCode >= 500 ? "Unexpected server error" : error.message,
    },
  });
}
~~~

Register it after routes. The next parameter must remain in the signature so Express recognizes error middleware.

## 10. Keep application creation separate from listening

src/app.js:

~~~javascript
import express from "express";
import { errorHandler } from "./middleware/error-handler.js";

export function createApp({ sessionController }) {
  const app = express();

  app.use(express.json({ limit: "100kb" }));
  app.post("/sessions", sessionController.create);

  app.use((request, response) => {
    response.status(404).json({
      error: {
        code: "ROUTE_NOT_FOUND",
        message: "Route not found",
      },
    });
  });

  app.use(errorHandler);
  return app;
}
~~~

src/server.js creates dependencies, calls createApp, and listens. Tests can import createApp without opening a real port.

## 11. REST is about resources and representations

Prefer:

~~~text
GET    /sessions
POST   /sessions
GET    /sessions/:id
PATCH  /sessions/:id
DELETE /sessions/:id
~~~

Avoid action-heavy paths such as /createSession or /deleteSession. HTTP methods already express the action.

## Real-world applications and edge cases

### Where this appears

- Versioned REST APIs used by mobile and browser clients
- Internal services sharing authentication and error middleware
- Multi-tenant applications where every request needs tenant context
- APIs that combine ordinary JSON routes with webhooks and streaming responses

### Edge cases to investigate

- Error-handling middleware is registered before routes and never sees their failures.
- JSON parsing fails before the controller, so route-level error handling cannot translate it.
- An async handler starts work but does not return or pass its rejected promise.
- Code sends a response and later calls next, causing a second handler to run.
- A broad parameter route such as /:id captures a more specific route registered later.
- A middleware trusts request.user or tenant headers before authentication establishes them.
- One route needs the raw request body for signature verification while global JSON parsing consumes it.
- An upload route buffers an unbounded file in memory.
- The application trusts forwarded headers from the public internet.
- A request-scoped object is accidentally stored globally and leaks data between users.

Middleware order is executable architecture. Document what each middleware establishes and which later handlers are allowed to rely on it.

## Pause and predict

- What happens if middleware neither sends a response nor calls next?
- Which layer should know response.status?
- Which layer should reject a negative duration?
- Why must the error handler be registered last?

## Guided practice

Rebuild every Lesson 09 route with Express. Add a Course resource:

- A course has id, title, and optional description.
- A session refers to an existing course ID.
- Deleting a course with sessions returns a deliberate conflict response.
- Query parameters filter sessions by course ID.

Write service tests using an in-memory repository before adding a database.

## Common mistakes

- Treating express.json as business validation
- Writing every rule in route callbacks
- Forgetting to pass async failures to error handling
- Returning sensitive internal errors
- Registering 404 or error middleware before routes
- Opening a server when app.js is imported
- Adding architecture layers with no responsibility
- Sending a response and then calling next

## Independent challenge

Create an API design document and implement:

- Course CRUD
- Study-session CRUD
- Pagination on collection endpoints
- Consistent validation errors
- A reusable not-found error
- A central error response shape

Explain one business rule that belongs in a service and one HTTP concern that belongs in a controller.

## Knowledge check

- What problem does middleware solve?
- How are route and query parameters different?
- Why separate app creation from listen?
- What does a repository isolate?
- How does Express identify error middleware?

## Explore with an AI agent

- Trace one request through my middleware, controller, service, and repository.
- Review whether my architecture boundaries own real responsibilities.
- Generate an API contract from my routes, then identify inconsistencies.
- Give me an Express bug caused by middleware order and let me diagnose it.

## Official reading

- [Express routing](https://expressjs.com/en/guide/routing.html)
- [Using Express middleware](https://expressjs.com/en/guide/using-middleware.html)
- [Express error handling](https://expressjs.com/en/guide/error-handling.html)
- [Express API reference](https://expressjs.com/en/api.html)
- [Node.js HTTP API underneath Express](https://nodejs.org/api/http.html)

## Definition of done

- Routes are thin and business rules remain reusable.
- Application creation does not open a port.
- 404 and error responses share a stable format.
- You can trace a request through every application boundary.
