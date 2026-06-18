# Lesson 24 - Capstone: Production-Style Study Tracker API

> Planning time: about 60 minutes  
> Building time: several focused sessions

## Outcome

Design, build, test, explain, and operate a complete Node.js service without depending on an AI agent to make every decision.

## The product

Build a Study Tracker API where authenticated users can:

- Register and log in
- Create courses
- Record study sessions
- View their own history
- Filter and paginate sessions
- View total minutes per course and date range
- Export their data as CSV
- Receive real-time session and report updates
- Log out and revoke the current token
- Receive fair service through documented rate limits

The API must prevent users from accessing one another's data. The final deployment runs several API instances behind a load balancer. Rate-limit counters and real-time event delivery must remain correct regardless of which instance receives a request.

## 1. Write the contract before implementation

Minimum endpoint set:

| Method | Path | Purpose | Success |
|---|---|---|---:|
| POST | /auth/register | Create user | 201 |
| POST | /auth/login | Create session token | 200 |
| POST | /auth/logout | Revoke session token | 204 |
| POST | /realtime/tickets | Create one-time WebSocket ticket | 201 |
| GET | /courses | List own courses | 200 |
| POST | /courses | Create course | 201 |
| GET | /courses/:id | Read own course | 200 |
| PATCH | /courses/:id | Update own course | 200 |
| DELETE | /courses/:id | Delete own course | 204 |
| GET | /sessions | Filter and paginate own sessions | 200 |
| POST | /sessions | Record study time | 201 |
| GET | /sessions/:id | Read own session | 200 |
| PATCH | /sessions/:id | Update own session | 200 |
| DELETE | /sessions/:id | Delete own session | 204 |
| GET | /reports/course-totals | Aggregate totals | 200 |
| GET | /exports/sessions.csv | Stream CSV | 200 |
| GET | /health | Liveness | 200 |
| GET | /ready | Readiness | 200 or 503 |

Document request fields, response shape, authentication, validation, and failure statuses for every endpoint.

Also document the WebSocket message protocol:

- Connection authentication and Origin policy
- session.created and session.deleted events
- Report progress events
- Acknowledgment and error envelopes
- Reconnection and resynchronization behavior

## 2. Model the data

Suggested tables:

### users

- id
- email, normalized and unique
- password_hash
- password_salt
- created_at

### auth_sessions

- id
- user_id
- token_hash, unique
- expires_at
- revoked_at
- created_at

### courses

- id
- user_id
- title
- description
- created_at
- updated_at

### study_sessions

- id
- course_id
- minutes with a positive check
- studied_at
- note
- created_at
- updated_at

### realtime_tickets

- id
- user_id
- ticket_hash, unique
- expires_at
- consumed_at
- created_at

Choose deletion behavior and uniqueness rules. Explain every foreign key and index.

## 3. Architecture target

~~~text
Clients
  -> load balancer
      -> Node.js instance A
      -> Node.js instance B
          -> PostgreSQL
          -> shared rate-limit store
          -> shared pub/sub or durable messaging

HTTP request inside an instance
  -> request context, rate limit, and authentication
  -> route and controller
  -> service and authorization rules
  -> repository, TypeORM, and PostgreSQL
  -> committed event
  -> shared messaging
  -> owning user's WebSocket on any instance
~~~

Errors travel back to one HTTP error boundary. Logs include request context but never secrets.

Suggested folders:

~~~text
src/
  app.js
  server.js
  config.js
  data-source.js
  entities/
  migrations/
  routes/
  controllers/
  services/
  repositories/
  middleware/
  errors/
  security/
  logging/
  rate-limit/
  realtime/
  messaging/
  workers/
test/
  unit/
  integration/
  http/
~~~

Adjust the structure if a simpler boundary explains the code better.

## Guided practice: implementation milestones

### Milestone A: foundation

- package.json and scripts
- Validated configuration
- App and server separation
- Error response format
- Health route

### Milestone B: database

- TypeORM EntitySchema definitions
- Reviewed migrations
- Constraints and indexes
- Repository integration tests

### Milestone C: identity

- Registration and password hashing
- Login and opaque tokens
- Authentication middleware
- Logout and expiration
- Ownership in every protected query
- Strict login and registration rate limits
- Shared limiter interface with deterministic tests

### Milestone D: product behavior

- Course operations
- Study-session operations
- Filtering and pagination
- Course total report
- Streaming CSV export

### Milestone E: real-time behavior

- One-time WebSocket tickets
- Origin validation and connection limits
- Versioned message envelopes
- Session and report events after database commit
- Heartbeats, reconnection, and slow-consumer policy
- Shared event distribution across instances

### Milestone F: horizontal scaling

- At least two identifiable Node.js instances
- Reverse proxy or load balancer
- Narrow Express proxy trust
- Shared rate-limit counters
- Shared WebSocket event delivery
- Readiness-based routing and connection draining

### Milestone G: quality

- Unit, integration, and HTTP tests
- Rate-limit boundary and store-failure tests
- WebSocket authentication, authorization, and message tests
- Structured logs and request IDs
- Readiness checks
- Graceful HTTP and WebSocket shutdown
- Secret redaction

### Milestone H: advanced feature

Choose two:

- Streaming CSV import
- Worker-based report generation
- Background job queue
- OpenAPI documentation
- Continuous integration
- Container-based local environment
- Durable event replay

## 5. Acceptance criteria

### Functional

- A user can complete the main journey from registration to report.
- Pagination and filters have documented defaults and limits.
- Invalid requests produce consistent field-level errors.
- Data persists across restarts.
- Connected clients receive authorized real-time updates.
- Reconnecting clients can recover missed state.

### Security

- Passwords use a salted password derivation function.
- Raw passwords and tokens never enter logs.
- Stored session tokens are hashed.
- Every private lookup includes owner scope.
- Request and body sizes are bounded.
- Login, general traffic, reports, connections, and messages have tested limits.
- Proxy-derived identity is trusted only through the documented topology.

### Data

- A fresh database can be built from migrations.
- Constraints protect core invariants.
- Transactions protect multi-step changes.
- Indexes correspond to real queries.

### Testing

- Service rules have unit tests.
- TypeORM repositories have PostgreSQL integration tests.
- Main API journeys have HTTP tests.
- Two-user isolation is explicitly tested.
- Rate limits are tested below, at, and above their thresholds.
- WebSocket tickets, Origin checks, message validation, and ownership are tested.
- A cross-instance test proves shared event delivery.
- Open handles do not keep the test process alive.

### Operations

- Logs are structured and correlated.
- Liveness and readiness differ.
- Startup fails safely.
- Shutdown drains HTTP and WebSocket work and closes resources.
- At least two instances operate behind a load balancer.
- Rate-limit counters and messaging are shared across instances.
- Instance, request, and connection behavior are observable.
- Deployment and rollback are documented.

## 6. Use AI without surrendering the learning

Good requests:

- Ask questions that help me choose between these two designs.
- Review this function for correctness and name the evidence.
- Give me a failing test scenario but not the implementation.
- Explain this official documentation section using my project.
- Ask me to explain my code before suggesting changes.

Less useful requests:

- Build the whole project for me.
- Fix everything.
- Add best practices.

The more specific your question, the more visible your own reasoning remains.

## 7. Architecture decision records

For important decisions, create a short record:

~~~text
Decision: Use opaque server-side session tokens.

Context:
We need immediate logout and simple revocation.

Options:
Opaque tokens, signed access tokens, external identity provider.

Choice:
Opaque tokens stored as hashes.

Consequences:
Every authenticated request performs a session lookup.
~~~

Write records for:

- Token strategy
- Course deletion behavior
- Repository boundary
- Rate-limit key, algorithm, and shared-store failure policy
- WebSocket recovery and message-delivery policy
- Load-balancing algorithm and proxy-trust topology
- Migration deployment
- One advanced feature

## 8. Final demonstration

Without opening source files first:

1. Explain the architecture.
2. Run all tests.
3. Start with a clean database.
4. Apply migrations.
5. Demonstrate the main user journey.
6. Attempt cross-user access.
7. Trigger validation and unexpected failures.
8. Inspect correlated logs.
9. Stream an export.
10. Exceed a documented HTTP and WebSocket limit.
11. Connect through the load balancer to one instance.
12. Commit an event on another instance and receive it in real time.
13. Mark one instance unready and verify traffic moves away.
14. Send a shutdown signal during active HTTP and WebSocket work.

Then open the code and trace one request end to end.

## Real-world applications and edge cases

The Study Tracker is intentionally small, but its architecture maps to larger products:

- Study sessions can become orders, tickets, workouts, deliveries, or financial entries.
- Course totals can become billing reports or analytics.
- Report progress can become media processing or data import status.
- User-owned resources can become tenant-owned business records.

Use this edge-case matrix during design and final review:

| Scenario | Expected system behavior |
|---|---|
| Mobile client retries a timed-out POST | Idempotency prevents duplicate creation |
| User crosses a timezone or daylight-saving boundary | Stored instants and display zones remain unambiguous |
| PostgreSQL becomes slow | Deadlines, pool bounds, readiness, and logs expose degradation |
| Shared limiter store fails | Documented fail-open or fail-closed policy applies |
| WebSocket disconnects during progress | Client reconnects and refetches durable job state |
| Event is delivered twice | Client or consumer deduplicates by event identity |
| One instance begins shutdown | Readiness removes it and active work drains |
| New and old versions overlap | Database and message formats remain compatible |
| User loses access while connected | Subscription authorization is refreshed or revoked |
| Export client disconnects | Cursor, stream, and temporary resources are released |
| Worker crashes after side effect | Job state and idempotency allow safe recovery |
| A tenant becomes unusually large | Pagination, indexes, quotas, and isolation still hold |

For every feature, ask: what if it happens twice, halfway, late, out of order, concurrently, after restart, or while a dependency is unavailable?

## Pause and predict

Before building, answer these architecture questions:

- If a controller calls TypeORM directly, which boundary from the target architecture disappears?
- If a course lookup uses only course ID, which security guarantee is missing?
- If a migration adds a required column immediately, what happens to existing rows?
- If report generation performs long CPU work in the route handler, which other endpoint is affected?
- If each instance keeps its own rate counters, which quota is actually enforced?
- If an event is published only to local sockets, which users miss it?
- If readiness remains successful during shutdown, what kind of race can occur?

Return to these answers after the final demonstration and correct anything your implementation taught you.

## Knowledge check: final review questions

- Why did you choose Node.js for this service?
- Where does validation happen, and why?
- Which invariants are protected by PostgreSQL?
- How does authentication differ from authorization here?
- Which query prevents cross-user access?
- What happens when a promise rejects in a controller?
- How do you know an instance is ready?
- What happens on SIGTERM?
- Which operation could block the event loop?
- How does a real-time event reach a socket on another instance?
- Which proxy headers does the application trust, and why?
- What happens when the shared limiter store fails?
- Which part would you redesign with twice the traffic?

If an answer is "because the AI suggested it," the decision is not yet yours.

## Common mistakes

- Building routes before writing the API contract
- Adding layers that have no clear responsibility
- Relying on application validation without database constraints
- Authenticating users but failing to scope resource queries by owner
- Testing only one user and the happy path
- Logging complete request bodies or authorization headers
- Loading a full export into memory before sending it
- Running CPU-heavy reports on the main event loop
- Using process-local rate counters in a multi-instance deployment
- Treating sticky sessions as a replacement for shared messaging
- Trusting arbitrary forwarded headers
- Ignoring dead or slow WebSocket clients
- Applying destructive schema changes without a compatibility plan
- Accepting AI-generated code that you cannot explain

## Independent challenge

After the required project works, change one major assumption:

- Support users in multiple time zones.
- Allow one course to be shared by several users.
- Add offline clients that may submit duplicate requests.
- Move report work to background jobs.

Write an architecture decision record, list the affected database constraints and API endpoints, add failing tests, and implement the smallest coherent change. The point is to discover how one product requirement travels through the whole system.

## 10. Completion rubric

Score each area from 0 through 3:

| Score | Meaning |
|---:|---|
| 0 | Missing |
| 1 | Works only in the happy path |
| 2 | Handles expected failures and is documented |
| 3 | Tested, explainable, and operationally deliberate |

Areas:

- API contract
- Architecture
- Database integrity
- Authentication
- Authorization
- Error handling
- Testing
- Logging
- Rate limiting
- Real-time messaging
- Streaming and worker behavior
- Load balancing
- Deployment

Aim for 2 everywhere before polishing anything to 3.

## Explore with an AI agent

- Interview me as the developer of this system and challenge vague answers.
- Review one architecture decision record for missing consequences.
- Generate adversarial acceptance tests from my API contract.
- Compare my final implementation with the official sources linked throughout the course.
- Ask me to diagnose a production scenario using only logs and symptoms.

## Primary reference index

- [Node.js API documentation](https://nodejs.org/api/)
- [Node.js Learn](https://nodejs.org/en/learn)
- [Express documentation](https://expressjs.com/)
- [PostgreSQL documentation](https://www.postgresql.org/docs/current/)
- [TypeORM documentation](https://typeorm.io/docs/)
- [npm documentation](https://docs.npmjs.com/)
- [ws documentation](https://github.com/websockets/ws)
- [Express rate limit documentation](https://express-rate-limit.mintlify.app/overview)
- [NGINX load balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)

## Definition of done

- A fresh clone can be configured, migrated, tested, and started from documentation.
- The complete user journey works.
- Cross-user access is blocked by design and tests.
- Limits remain global across several instances.
- Real-time updates reach authorized users across instance boundaries.
- Traffic drains safely through the load balancer.
- No secret is committed or logged.
- Production lifecycle behavior is observable and documented.
- You can explain the important design choices in your own words.
