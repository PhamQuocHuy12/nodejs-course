# Full-Stack JavaScript Interview Questions and Short Answers

This guide focuses on the core requirements in the job description: JavaScript, a modern frontend framework, Node.js, relational and NoSQL databases, build tooling, and Git workflows. Adapt experience-based answers to match projects you actually worked on.

## JavaScript fundamentals

### 1. Explain the differences between `var`, `let`, and `const`.

`var` is function-scoped, can be redeclared, and is hoisted with an initial value of `undefined`. `let` and `const` are block-scoped and remain in the temporal dead zone until declared. `const` prevents reassignment, but objects stored in a `const` can still be mutated.

### 2. What is hoisting, and how does it affect variables and functions?

JavaScript processes declarations before executing their scope. Function declarations can be called before their source line; `var` is hoisted and initialized to `undefined`; `let`, `const`, and classes are hoisted but cannot be accessed before their declarations.

### 3. What is a closure? Give a practical example.

A closure is a function that retains access to variables from its lexical scope after the outer function has finished. It is useful for private state, factories, callbacks, and memoization—for example, a `createCounter` function returning a function that updates a private count.

### 4. Explain JavaScript’s event loop.

The event loop lets JavaScript coordinate asynchronous work while running one main call stack. When the stack is empty, it processes queued microtasks first and then eligible macrotasks, allowing callbacks to run without blocking while I/O is handled elsewhere.

### 5. What is the difference between microtasks and macrotasks?

Promise callbacks and `queueMicrotask` use the microtask queue; timers, I/O callbacks, and UI events generally use task or macrotask queues. After the current stack completes, all queued microtasks run before the next macrotask.

### 6. How do Promises work?

A Promise represents an asynchronous result and has `pending`, `fulfilled`, or `rejected` state. Consumers attach `then`, `catch`, or `finally` handlers, and returned values or Promises allow operations to be chained without deeply nested callbacks.

### 7. Compare callbacks, Promises, and `async/await`.

Callbacks are simple but can become hard to compose and handle errors with. Promises provide chaining and centralized rejection handling. `async/await` is syntax built on Promises that usually makes sequential asynchronous code easier to read.

### 8. How do you handle errors in asynchronous code?

With `async/await`, wrap awaited operations in `try/catch` and either recover or rethrow a meaningful error. With Promises, return the chain and use `.catch()`. Avoid empty catches and unhandled rejections, and add useful context without exposing secrets.

### 9. What is the difference between `==` and `===`?

`==` performs type coercion before comparison, which can produce surprising results such as `0 == false`. `===` compares both type and value without coercion, so it is the safer default.

### 10. Explain `null`, `undefined`, and undeclared variables.

`undefined` normally means a value has not been assigned or is missing. `null` is an explicit empty value. An undeclared variable has no binding at all and accessing it normally throws `ReferenceError`.

### 11. How does the `this` keyword work in JavaScript?

For regular functions, `this` is determined by how the function is called: as a method, with `new`, through `call`/`apply`/`bind`, or as a plain function. Arrow functions do not bind their own `this`; they capture it lexically.

### 12. How are arrow functions different from regular functions?

Arrow functions have lexical `this`, `arguments`, and `super`, and they cannot be used as constructors. Regular functions receive `this` from the call site and are usually preferable for object methods that need a dynamic receiver.

### 13. Explain prototypal inheritance.

Objects can delegate property lookup to another object through their prototype chain. JavaScript classes provide cleaner syntax, but methods are still normally stored on prototypes and shared by instances.

### 14. What is the difference between shallow and deep copying?

A shallow copy duplicates only the top-level container, so nested objects remain shared references. A deep copy recursively duplicates supported nested values. `structuredClone` supports many built-in types, while JSON serialization has data-loss limitations.

### 15. Explain destructuring, rest parameters, and spread syntax.

Destructuring extracts values from objects or arrays. Rest collects remaining values, such as `function f(...args)`, while spread expands an iterable or object, such as `[...items]` or `{...user}`.

### 16. What are JavaScript modules? Compare CommonJS and ES modules.

Modules organize code into explicit imports and exports. CommonJS uses `require` and `module.exports` and traditionally loads synchronously; ES modules use `import` and `export`, are statically analyzable, and support optimizations such as tree shaking.

### 17. Explain mutable and immutable data.

Mutable data can be changed in place; immutable updates produce a new value instead. Immutable patterns make state changes easier to compare, debug, and reason about, especially in frontend state management, though they can add allocation overhead.

### 18. How do `map`, `filter`, and `reduce` differ?

`map` transforms every array element, `filter` keeps elements that satisfy a condition, and `reduce` combines elements into one accumulated result. Use the method that most clearly expresses the intended operation.

### 19. What causes memory leaks in JavaScript applications?

Common causes include forgotten event listeners or timers, unbounded caches, global references, retained closures, and subscriptions that are never cleaned up. Heap snapshots and allocation profiling can reveal which objects remain reachable and why.

### 20. How would you debounce or throttle a function?

Debouncing waits until calls stop for a chosen interval, which is useful for search input. Throttling limits execution to at most once per interval, which is useful for scroll or resize handlers. Both can be implemented with timers and should define leading/trailing behavior.

## Frontend framework experience

### 21. Which frontend framework have you used most, and for how long?

Give an honest duration, versions used, and project scope. For example: “I have used React professionally for about three years, mainly building TypeScript dashboards with React Router, React Query, and component tests.”

### 22. Describe the architecture of your most recent frontend project.

Explain the application by feature areas, shared UI components, routing, server-state access, client state, forms, and tests. Mention why the structure was chosen and one trade-off or improvement you made.

### 23. How do components communicate in your framework?

Typically, parents pass data down through props or inputs and children send events or callbacks upward. Context, dependency injection, stores, or services can support cross-tree communication, but should not replace simple local data flow unnecessarily.

### 24. Explain component lifecycle in your framework.

Components are created or mounted, updated when inputs or state change, and eventually destroyed or unmounted. Lifecycle hooks or effects are used for side effects, with cleanup for subscriptions, listeners, timers, and unfinished requests.

### 25. How do you manage local and global state?

Keep temporary UI state close to the component and use a shared store or context only for state needed across features. Treat remotely fetched data as server state and use a query/cache library when appropriate.

### 26. When should state remain local rather than global?

State should remain local when only one component or a small subtree needs it, such as whether a menu is open. This reduces coupling, prevents unrelated updates, and makes ownership easier to understand.

### 27. How do you pass data from a parent component to a child?

Use the framework’s one-way input mechanism—for example, React props, Angular `@Input`, or Vue props. Keep the input contract typed and avoid letting the child silently mutate parent-owned data.

### 28. How can a child component notify its parent?

The parent supplies a callback, or the child emits an event using the framework’s output mechanism. The child describes what happened, while the parent decides how application state changes.

### 29. How do you share data between unrelated components?

Lift state to their nearest common owner, or use a scoped context, service, or store when the components are far apart. For API data, a shared query cache may remove the need for a separate global store.

### 30. What causes a component to re-render?

Common triggers are state changes, new props or inputs, changes in consumed context/store values, and parent rendering. Exact behavior varies by framework, so profiling should be used before attempting optimization.

### 31. How do you prevent unnecessary re-renders?

Keep state scoped narrowly, avoid recreating expensive values unnecessarily, use stable keys, and memoize only where profiling shows value. Also split large components and subscribe to the smallest possible portion of shared state.

### 32. How do you fetch and display API data?

Call the API through a dedicated client or service, validate or type the response, and expose explicit loading, success, empty, and error states. A query library can provide caching, request deduplication, retries, and invalidation.

### 33. How do you represent loading, success, empty, and error states?

Model each state explicitly so the UI never mistakes “not loaded” for “empty.” Show an appropriate loader, the data, a useful empty-state action, or an error message with retry when retrying is safe.

### 34. How do you cancel an API request when a component is destroyed?

Use `AbortController` with `fetch`, or the cancellation/unsubscription feature of the HTTP library or observable. Trigger cleanup in the framework’s unmount or destruction lifecycle so stale responses do not update abandoned UI.

### 35. How do you implement client-side routing?

Define a route tree mapping URLs to page components, including nested layouts and a not-found route. Use route parameters for resource identity, query parameters for shareable filters, and lazy loading for larger feature bundles.

### 36. How do you protect authenticated routes?

Use a route guard or wrapper to check authentication and redirect unauthenticated users. This improves UX only; the backend must still validate the user and authorization for every protected operation.

### 37. How do you build and validate forms?

Use controlled/model-bound fields or a form library, keep a clear form schema, and validate both individual fields and cross-field rules. Show accessible errors near their fields and repeat validation on the server.

### 38. How do you handle server-side validation errors in a form?

Map structured field errors from the API to the relevant inputs and show general errors at form level. Preserve safe user input, focus the first invalid field where helpful, and do not expose raw server internals.

### 39. How do you create reusable components without over-engineering?

Start with a concrete use case and extract a shared component only when a stable pattern appears. Prefer composition and a small, typed API over many boolean options that attempt to predict every future use.

### 40. How do you implement lazy loading or code splitting?

Split bundles at meaningful boundaries such as routes or large optional features using dynamic imports and the framework’s lazy-loading API. Provide a loading fallback and measure bundle and navigation performance to verify the benefit.

### 41. How do you improve the performance of a slow frontend page?

Measure first with browser and framework profilers. Then address the actual bottleneck—for example, reduce bundle size, virtualize long lists, cache requests, optimize images, avoid repeated calculations, or reduce unnecessary rendering.

### 42. How do you diagnose a frontend memory leak?

Reproduce the suspected flow repeatedly and compare browser heap snapshots or allocation timelines. Inspect retained objects and their reference paths, then check listeners, timers, subscriptions, caches, detached DOM nodes, and closures.

### 43. What is the virtual DOM, and why is it useful?

It is an in-memory representation of UI used by frameworks such as React and Vue. The framework compares representations and applies necessary DOM changes, giving developers a declarative model; it is not automatically faster in every situation.

### 44. How do you safely render user-generated content?

Render it as text by default so the framework escapes markup. If HTML must be supported, sanitize it with a trusted, maintained allowlist-based library and avoid unsafe raw-HTML APIs unless the content has been sanitized.

### 45. What is cross-site scripting, and how do you prevent it?

XSS occurs when attacker-controlled content executes as script in another user’s browser. Prevent it through contextual output escaping, sanitizing permitted HTML, avoiding unsafe DOM APIs, using a restrictive Content Security Policy, and protecting sensitive tokens.

### 46. How do you store authentication tokens securely?

For browser applications, short-lived credentials and `Secure`, `HttpOnly`, appropriately configured `SameSite` cookies are often safer because JavaScript cannot read them. If cookies authenticate requests, implement CSRF defenses; avoid storing long-lived sensitive tokens in `localStorage`.

### 47. How do you test frontend components?

Test behavior visible to the user: render the component, interact through accessible controls, and assert output. Mock only external boundaries such as the network, and use a smaller number of end-to-end tests for critical flows.

### 48. Describe a difficult frontend bug you diagnosed.

Answer with Situation, Task, Action, and Result. Name the symptom, evidence gathered, tools used, root cause, fix, verification, and preventive change such as a regression test or monitoring.

### 49. How have you handled a major framework or dependency upgrade?

Review migration guides and release notes, upgrade in small stages, run automated tests, and manually verify high-risk flows. Isolate breaking changes in a dedicated branch and update CI, types, and build configuration as needed.

### 50. What would you improve in the architecture of your last project?

Choose a real, non-destructive improvement—for example, clearer feature boundaries, fewer global dependencies, better API typing, or stronger tests. Explain its expected impact, cost, risks, and how you would migrate incrementally.

## Node.js and backend development

### 51. Describe your professional Node.js experience.

State the honest duration and provide scale and responsibilities: framework, API types, databases, authentication, testing, deployment, and operational support. Include one measurable result or difficult problem you owned.

### 52. Why is Node.js suitable for I/O-intensive applications?

Node.js uses non-blocking I/O and an event-driven model, so one process can keep many network or file operations in flight without a thread per request. This makes it effective for APIs, proxies, streaming, and real-time services.

### 53. When is Node.js a poor choice?

CPU-heavy work on the main thread can block every request, so intensive image processing, scientific computation, or large synchronous transformations may fit another runtime or a worker/job architecture better. Node can still coordinate such work through worker threads or separate services.

### 54. How does the Node.js event loop work?

JavaScript callbacks run on the main thread while the operating system and libuv handle supported asynchronous work. The loop moves through phases for timers, polling, checks, and callbacks, while Promise and next-tick queues have special priority.

### 55. If Node.js is single-threaded, how does it handle concurrent requests?

JavaScript normally runs on one event-loop thread, but asynchronous I/O is delegated to the OS or libuv thread pool. Completed operations queue callbacks, allowing many requests to progress concurrently without executing JavaScript simultaneously on the main thread.

### 56. What operations can block the event loop?

Large synchronous loops, synchronous filesystem or crypto calls, huge JSON parsing, inefficient regular expressions, compression, and CPU-intensive transformations can block it. Break up, stream, offload, or move such work to worker threads or background services.

### 57. How would you diagnose a blocked or slow event loop?

Correlate latency and resource metrics, measure event-loop delay, and capture CPU profiles or flame graphs under representative load. Inspect long synchronous stacks, garbage collection, slow dependencies, database queries, and pool saturation.

### 58. Compare CommonJS with ES modules in Node.js.

CommonJS uses `require` and `module.exports`, while ESM uses static `import` and `export`. ESM supports static analysis and top-level `await`, but migration requires attention to package type, file extensions, path handling, and library compatibility.

### 59. How do you structure a maintainable Node.js application?

Organize by business feature or bounded module, with clear HTTP, application, domain, and data-access responsibilities. Keep controllers thin, inject external dependencies, centralize cross-cutting concerns, and test important business rules independently.

### 60. What are controllers, services, repositories, and middleware?

Controllers translate HTTP input and output, services coordinate business use cases, and repositories abstract persistence operations. Middleware handles request-wide concerns such as authentication, correlation IDs, logging, and rate limiting.

### 61. Describe the request lifecycle in NestJS, AdonisJS, or your preferred framework.

In NestJS, a request typically passes middleware, guards, interceptors, pipes, the controller, and service logic; exception filters handle failures. Mention that exact ordering matters because authentication, validation, transformation, and response wrapping happen at different stages.

### 62. How do you design a REST API?

Model endpoints around resources, use HTTP methods consistently, validate inputs, return meaningful status codes, and define a predictable error format. Include authentication, authorization, pagination, idempotency, versioning strategy, documentation, and observability.

### 63. Which HTTP methods and status codes do you commonly use?

Use `GET` to read, `POST` to create or trigger non-idempotent actions, `PUT` to replace, `PATCH` to update partially, and `DELETE` to remove. Common statuses include `200`, `201`, `204`, `400`, `401`, `403`, `404`, `409`, `422`, and `500`.

### 64. What is the difference between `PUT` and `PATCH`?

`PUT` conventionally replaces the full representation at a URI and should be idempotent. `PATCH` applies a partial change and may use formats such as JSON Merge Patch; its idempotency depends on the operation design.

### 65. How do you validate and sanitize request data?

Define schemas or DTOs at the system boundary, reject unknown or invalid fields as appropriate, and convert types explicitly. Validation checks correctness; sanitization safely normalizes permitted content, while parameterized queries and output escaping address injection risks.

### 66. How do you implement centralized error handling?

Let application code throw typed or classified errors and convert them in one error middleware/filter into a consistent response. Log internal details with a request ID, return safe messages, and distinguish expected client errors from unexpected server faults.

### 67. How do you implement authentication and authorization?

First verify identity using a session, token, or identity provider; then enforce permissions for the requested action and resource. Keep authorization server-side, deny by default, and use roles, permissions, policies, or ownership checks as the domain requires.

### 68. What is the difference between session-based and token-based authentication?

With sessions, the server stores session state and the client sends an opaque session ID. With self-contained tokens, the server validates signed claims without necessarily looking up session state, which scales differently but makes revocation and token protection more complex.

### 69. How do access tokens and refresh tokens work?

An access token is short-lived and sent to protected APIs. A refresh token is longer-lived, more carefully stored, and exchanged for new access tokens; rotating refresh tokens and detecting reuse reduce the damage from theft.

### 70. How should passwords be stored?

Store a unique salted password hash using a password-specific algorithm such as Argon2id, bcrypt, or scrypt with reviewed cost settings. Never store reversible passwords; also apply rate limits, secure reset flows, and rehash older hashes when settings improve.

### 71. How do you prevent SQL injection and NoSQL injection?

Use parameterized queries or safe ORM APIs and never construct queries by concatenating untrusted input. Validate field names and operators against an allowlist, especially for dynamic sorting, filters, and MongoDB-style query objects.

### 72. How do you implement logging without exposing sensitive information?

Use structured logs with severity, timestamps, service context, and correlation IDs. Redact passwords, tokens, cookies, personal data, and secrets at a central boundary, and define retention and access controls.

### 73. How do you manage environment variables and secrets?

Validate configuration at startup and fail fast when required values are missing. Keep secrets in a managed secret store or protected runtime environment, never commit them, grant least privilege, rotate them, and avoid printing them in logs.

### 74. How do you implement pagination, filtering, and sorting?

Validate all query parameters, allowlist filter and sort fields, impose a maximum page size, and use matching indexes. Offset pagination is simple; cursor/keyset pagination is more stable and efficient for large or frequently changing datasets.

### 75. How do you handle file uploads safely?

Enforce size and count limits, inspect content rather than trusting extensions, generate server-side filenames, and store files outside executable application paths. Use access control, malware scanning where appropriate, and direct object-storage uploads for large files.

### 76. How do you implement rate limiting?

Choose a policy such as token bucket or sliding window, identify clients carefully, and store shared counters in a system such as Redis for multiple instances. Return `429 Too Many Requests` and consider separate limits for login and expensive endpoints.

### 77. How do you process long-running or background jobs?

Put a small, durable job message on a queue and let workers process it outside the request lifecycle. Make jobs idempotent, define retries with backoff, use dead-letter handling, record status, and propagate correlation information.

### 78. How do you test a Node.js API?

Unit-test business logic, integration-test database and infrastructure boundaries, and send HTTP requests against a test application for endpoint behavior. Use isolated test data, deterministic setup, and assertions for authorization, validation, failure cases, and side effects.

### 79. How do unit, integration, and end-to-end tests differ?

Unit tests isolate a small unit and are fast; integration tests verify real collaboration between selected components; end-to-end tests exercise the deployed-style system through its public interface. A healthy suite uses many focused tests and fewer expensive full-system tests.

### 80. Describe a production backend issue you diagnosed and resolved.

Use a concrete STAR example. Explain impact and detection, the evidence and hypothesis process, safe mitigation, root-cause fix, verification, and the monitoring or regression test added afterward.

## Relational and NoSQL databases

### 81. Compare relational and NoSQL databases.

Relational databases provide structured schemas, joins, constraints, and strong transaction support. NoSQL is a broad category optimized around models such as documents or key-value access, often favoring flexible records or predictable horizontal access patterns. The choice should follow data relationships and query needs.

### 82. When would you choose PostgreSQL/MySQL over MongoDB/DynamoDB?

Choose a relational database when relationships, joins, constraints, and multi-record transactions are central. Choose a document or key-value database when aggregates fit that model and access patterns, scale, latency, or schema flexibility justify its trade-offs.

### 83. Explain primary keys, foreign keys, and unique constraints.

A primary key uniquely identifies each row. A foreign key enforces that a referenced row exists and defines referential behavior. A unique constraint prevents duplicate values or value combinations beyond the primary key.

### 84. What is database normalization?

Normalization organizes relational data to reduce duplication and update anomalies, commonly by separating entities and representing relationships with keys. Third normal form generally means non-key attributes depend on the key, the whole key, and not other non-key attributes.

### 85. When might denormalization be appropriate?

Denormalization can reduce expensive joins or precompute read-heavy views when measured performance needs justify it. It introduces duplication, so the design needs a reliable update strategy and a clear source of truth.

### 86. Explain `INNER JOIN` and `LEFT JOIN`.

`INNER JOIN` returns only rows with matching records on both sides. `LEFT JOIN` keeps every row from the left table and fills right-side columns with `NULL` when no match exists.

### 87. What are database indexes, and how do they improve performance?

An index is an auxiliary data structure, often a B-tree, that helps the database locate or order rows without scanning the whole table. Good indexes reflect real filter, join, and sort patterns, including column order in composite indexes.

### 88. What are the costs or disadvantages of adding indexes?

Indexes consume storage and memory, and every insert, update, or delete may need to maintain them. Redundant or poorly chosen indexes can slow writes and complicate planning without improving important queries.

### 89. How would you diagnose a slow SQL query?

Capture the real query and parameters, inspect its execution plan with tools such as `EXPLAIN ANALYZE`, and check scans, row estimates, joins, sorts, locks, and I/O. Then improve the query, indexes, statistics, or data model and measure again with representative data.

### 90. What is a database transaction, and when should one be used?

A transaction groups operations into one logical unit that commits completely or rolls back. Use one when partial success would violate a business invariant, such as moving funds between accounts or creating an order and its lines.

### 91. Explain the ACID properties.

Atomicity means all-or-nothing; consistency means committed work preserves defined constraints; isolation controls interference between concurrent transactions; durability means committed data survives failures according to the database guarantee.

### 92. What are transaction isolation levels?

Isolation levels trade concurrency for protection against anomalies such as dirty reads, non-repeatable reads, and phantoms. Standard levels are Read Uncommitted, Read Committed, Repeatable Read, and Serializable, though exact behavior depends on the database implementation.

### 93. What is an ORM, and what are its advantages and disadvantages?

An ORM maps application objects or models to database operations, providing productivity, migrations, parameterization, and type support. It can hide inefficient queries or database-specific features, so developers still need SQL knowledge and query-plan awareness.

### 94. How do you safely run database schema migrations?

Keep migrations versioned, tested, reviewed, observable, and compatible with the application rollout. For risky changes, use expand-and-contract: add compatible structures, backfill safely, switch code, then remove old structures later.

### 95. How do you model one-to-one, one-to-many, and many-to-many relationships?

One-to-one usually uses a foreign key with a unique constraint; one-to-many puts a foreign key on the “many” table; many-to-many uses a junction table containing both foreign keys. Add constraints and indexes based on cardinality and queries.

### 96. How would you model data in MongoDB: embedding or referencing?

Embed data that is bounded, usually read together, and owned by one aggregate. Reference data that grows without bound, is shared, changes independently, or is queried separately. Base the choice on access patterns and document-size limits.

### 97. What is an aggregation pipeline in MongoDB?

It processes documents through ordered stages such as `$match`, `$project`, `$group`, `$sort`, and `$lookup`. Filter early, project only needed fields, and create indexes that support the initial matching and sorting stages.

### 98. How do you implement pagination efficiently on a large dataset?

Prefer keyset/cursor pagination using a stable, indexed, unique ordering such as `(created_at, id)`. Query records after the last cursor and fetch one extra to determine whether another page exists, avoiding the growing cost and instability of large offsets.

### 99. How do you prevent race conditions when multiple requests update the same record?

Use atomic conditional updates, transactions with appropriate locking, optimistic concurrency through version fields, or database constraints. Select the strategy based on contention, and retry only operations designed to be safely repeatable.

### 100. Describe a database performance or data-consistency problem you solved.

Use a genuine STAR story. Include the query or invariant involved, measurements or execution-plan evidence, the implemented index/query/transaction change, the resulting improvement, and how you prevented regression.

## Additional must-have topics to review

The original 100 questions focus on JavaScript, frontend, Node.js, and databases. Because the job description also explicitly requires Git/GitFlow and mentions build tools, prepare these short follow-ups too.

### Git and GitFlow

- **What is the difference between merge and rebase?** Merge combines histories with a merge commit; rebase replays commits onto a new base for a linear history. Do not rebase shared history unless the team explicitly coordinates it.
- **How do you resolve a merge conflict?** Understand both intended changes, edit the file into the correct combined state, run tests, stage the resolution, and continue the merge or rebase.
- **What is GitFlow?** It is a branching model using long-lived `main` and `develop` branches plus feature, release, and hotfix branches. It supports scheduled releases but may be heavier than trunk-based development.
- **How do you undo a production commit safely?** Prefer `git revert` on a shared branch because it adds a new inverse commit without rewriting published history.
- **What makes a good pull request?** A focused change, clear purpose and testing notes, manageable size, passing checks, and context for reviewers; avoid mixing unrelated refactoring with functional changes.

### Yarn, Webpack, and Gulp

- **What does a lockfile do?** It records exact resolved dependency versions and integrity data so installations are reproducible; it should normally be committed.
- **What does Webpack do?** It builds a dependency graph from application entry points, transforms modules through loaders, and emits optimized bundles using plugins and configuration.
- **What is tree shaking?** It removes exports that can be proven unused, working best with statically analyzable ES modules and side-effect-aware packages.
- **What are source maps?** They map generated or minified code back to original source for debugging; production access should be configured so source code is not exposed unintentionally.
- **When would you use Gulp?** Gulp is a programmable task runner suited to custom pipelines such as copying assets or transforming files, although many modern projects use package scripts and bundler plugins instead.

## Final preparation tip

For experience questions, avoid memorizing generic answers. Prepare four or five true project stories covering a difficult bug, a performance improvement, a production incident, an architectural decision, and a disagreement or trade-off. Explain each using **Situation → Task → Action → Result**, and be ready for follow-up questions about what *you* personally did.
