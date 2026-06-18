# Lesson 14 - Automated Testing

> Reading time: about 32 minutes  
> Practice time: about 90 minutes

## Outcome

Write focused unit, integration, and HTTP tests with the built-in node:test runner, isolate dependencies, and use tests to guide behavior rather than freeze implementation details.

## Why tests matter

Manual testing answers, “Does this example work right now?” Automated testing answers, “Does this behavior still work after the next change?”

Tests are executable evidence, not proof that no bug exists. A valuable suite covers important behavior, failure paths, and boundaries.

## 1. Your first built-in test

~~~javascript
import assert from "node:assert/strict";
import test from "node:test";

test("adds study minutes", () => {
  const result = 25 + 40;
  assert.equal(result, 65);
});
~~~

Run:

~~~powershell
node --test
~~~

The test runner discovers supported test files. The strict assertion module avoids surprising loose comparisons.

## 2. Arrange, act, assert

A readable test has three conceptual steps:

~~~javascript
test("calculates total minutes", () => {
  // Arrange
  const sessions = [
    { minutes: 25 },
    { minutes: 40 },
  ];

  // Act
  const result = calculateTotal(sessions);

  // Assert
  assert.equal(result, 65);
});
~~~

Comments are optional when the structure is obvious. One test should communicate one meaningful behavior.

## 3. Test outcomes, not private steps

Suppose calculateTotal changes from reduce to a loop. A behavior test should remain valid because callers still receive the same result.

Fragile tests inspect private helper calls or duplicate implementation details. They punish refactoring without protecting user behavior.

## 4. Test expected errors

~~~javascript
test("rejects a nonpositive duration", () => {
  assert.throws(
    () => validateDuration(0),
    {
      name: "ValidationError",
      message: "Minutes must be a positive integer",
    },
  );
});
~~~

For promises:

~~~javascript
test("rejects an unknown course", async () => {
  await assert.rejects(
    () => service.createSession({
      courseId: 999,
      minutes: 25,
    }),
    {
      name: "NotFoundError",
    },
  );
});
~~~

Await assert.rejects. Otherwise the test can finish before the rejection is examined.

## 5. Unit-test services with dependency injection

~~~javascript
test("saves a valid session", async () => {
  const savedInputs = [];

  const repository = {
    async create(input) {
      savedInputs.push(input);
      return { id: 1, ...input };
    },
  };

  const service = createSessionService(repository);

  const result = await service.create({
    course: "Node.js",
    minutes: 25,
  });

  assert.equal(result.id, 1);
  assert.deepEqual(savedInputs, [
    { course: "Node.js", minutes: 25 },
  ]);
});
~~~

The fake repository is small and controlled. The test does not need PostgreSQL to verify a service rule.

## 6. Integration tests exercise real boundaries

A TypeORM repository test should use a real test database because database behavior includes:

- SQL generation
- Constraints
- Transactions
- Data types
- Relationship mapping

Good isolation options:

- Give each test transaction and roll it back.
- Truncate known tables between tests.
- Create a temporary schema or database per test worker.

Never point tests at development or production data. Validate the database name before destructive cleanup.

## 7. Test an HTTP endpoint

Because createApp does not listen during import, a test can open an ephemeral port:

~~~javascript
import assert from "node:assert/strict";
import { after, test } from "node:test";
import { createApp } from "../src/app.js";

const app = createApp(createTestDependencies());
const server = app.listen(0);

after(() => {
  return new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) reject(error);
      else resolve();
    });
  });
});

test("GET /health reports ok", async () => {
  const address = server.address();
  const url = "http://127.0.0.1:" + address.port + "/health";
  const response = await fetch(url);

  assert.equal(response.status, 200);
  assert.deepEqual(await response.json(), { status: "ok" });
});
~~~

Port 0 asks the operating system for an available port. This avoids assuming 3000 is free.

## 8. The testing pyramid is a cost model

- Unit tests are fast and local.
- Integration tests provide stronger boundary confidence at higher setup cost.
- End-to-end tests cover a full flow but are slower and harder to diagnose.

Use many focused tests and enough larger tests to prove the pieces truly work together.

## 9. Red, green, refactor

1. Red: write a failing test for the desired behavior.
2. Green: make the smallest reasonable implementation pass.
3. Refactor: improve the design while all tests remain green.

The first failure matters. A test that passes before implementation may not test the new behavior.

## 10. Determinism

Flaky tests often depend on:

- Current clock time
- Random values
- Test order
- Shared database rows
- External network services
- Fixed ports
- Unawaited asynchronous work

Inject a clock or ID generator when exact control matters.

~~~javascript
function createSession(input, now = () => new Date()) {
  return {
    ...input,
    studiedAt: now(),
  };
}
~~~

## Real-world applications and edge cases

### Where this appears

- Pull requests run fast unit tests before merge.
- Deployment pipelines run migrations and integration tests.
- Contract tests protect mobile clients released on slower schedules.
- Regression tests preserve behavior discovered during an incident.

### Edge cases to investigate

- A test passes alone but fails in the suite because global state leaked.
- Parallel tests truncate the same database tables.
- A test depends on local timezone, locale, or current daylight-saving transition.
- Fake timers are enabled but a promise or I/O callback still uses real scheduling.
- A mocked repository always succeeds, so transaction and constraint failures remain untested.
- An assertion finishes before background async work rejects.
- A fixed port is already used by another test process.
- Random fixtures occasionally create duplicate values.
- Tests call a real external service and become slow, costly, or nondeterministic.
- High coverage hides missing assertions or untested failure behavior.

When production finds a defect, first create the smallest repeatable test that demonstrates it. Then fix the cause and keep the test as institutional memory.

## Pause and predict

- Should changing reduce to a loop break a total-minutes test?
- Why is a fake repository appropriate for a service unit test?
- Why is it insufficient for testing TypeORM mappings?
- What does port 0 solve?

## Guided practice

Add tests in this order:

1. Unit tests for validation and service rules.
2. Integration tests for TypeORM repositories and constraints.
3. HTTP tests for status, headers, and response bodies.
4. Authentication tests for missing, invalid, expired, and valid tokens.
5. Ownership tests for two different users.
6. Rate-limit tests below, at, and above each threshold.
7. Shared-store failure tests using a fake clock and store.

For every successful route, write at least one likely failure test.

## Common mistakes

- Testing only happy paths
- Forgetting to await async assertions
- Sharing mutable state between tests
- Mocking so much that no real application behavior remains
- Using a production database
- Depending on exact internal helper calls
- Leaving servers or database connections open
- Treating coverage percentage as a quality score

## Independent challenge

Use red, green, refactor to add a rule: a study session cannot exceed 12 hours.

Include:

- Service unit test
- Database constraint or clear justification for not adding one
- API response test
- Boundary cases at exactly 720 and 721 minutes

Write a sentence explaining what each test level protects.

## Knowledge check

- How do unit and integration tests differ?
- What makes a test deterministic?
- Why should tests focus on public behavior?
- What does dependency injection make easier?
- What does the first red failure tell you?

## Explore with an AI agent

- Review my test names and tell me which behaviors are unclear.
- Identify tests coupled to implementation details without rewriting them.
- Generate edge cases for one API rule, then let me choose the valuable cases.
- Diagnose an open-handle test failure from evidence I provide.

## Official reading

- [Node.js test runner](https://nodejs.org/api/test.html)
- [Strict assertions](https://nodejs.org/api/assert.html#strict-assertion-mode)
- [Testing Node.js applications](https://nodejs.org/en/learn/test-runner/using-test-runner)
- [Testing coverage](https://nodejs.org/en/learn/test-runner/collecting-code-coverage)

## Definition of done

- One command runs all tests.
- Service tests do not require a database.
- Repository tests use isolated real PostgreSQL behavior.
- HTTP tests verify success and failure responses.
- Every opened server and connection is closed.
