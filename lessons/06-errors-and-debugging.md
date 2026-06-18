# Lesson 06 - Errors and Debugging

> Reading time: about 25 minutes  
> Practice time: about 45 minutes

## Outcome

Create useful errors, preserve their context, handle expected failures at the right boundary, and use stack traces and the debugger to locate defects.

## 1. An Error is structured failure information

~~~javascript
const error = new Error("Course name is required");

console.log(error.name);
console.log(error.message);
console.log(error.stack);
~~~

An Error contains a type name, human-readable message, and stack trace. Throw Error objects rather than strings.

## 2. Throwing changes control flow

~~~javascript
function parseDuration(value) {
  const duration = Number(value);

  if (!Number.isInteger(duration) || duration <= 0) {
    throw new TypeError("Duration must be a positive integer");
  }

  return duration;
}
~~~

The function either returns a valid duration or throws. It does not return a confusing mixture of valid values, null, false, and error text.

Handle it at a boundary that can make a useful decision:

~~~javascript
try {
  const duration = parseDuration(process.argv[2]);
  console.log("Duration:", duration);
} catch (error) {
  console.error(error.message);
  process.exitCode = 1;
}
~~~

## 3. Rejected promises are asynchronous errors

~~~javascript
import { readFile } from "node:fs/promises";

async function loadSettings() {
  try {
    return await readFile("settings.json", "utf8");
  } catch (error) {
    throw new Error("Could not load application settings", {
      cause: error,
    });
  }
}
~~~

The new error adds application context. The cause preserves the original failure for debugging.

The caller must await or return the promise for its rejection to remain connected to control flow.

## 4. Expected versus unexpected failures

Operational failures can happen even when the program is correct:

- A requested record does not exist
- Input fails validation
- A file is missing
- A database connection times out

Programmer errors indicate a defect:

- Reading a property from undefined
- Calling the wrong function
- Breaking an internal invariant

Expected failures often become a controlled response. Programmer errors should be logged with context and fixed, not disguised as validation messages.

## 5. Custom error classes communicate categories

~~~javascript
export class ValidationError extends Error {
  constructor(message, details = []) {
    super(message);
    this.name = "ValidationError";
    this.details = details;
  }
}
~~~

Use:

~~~javascript
if (!course.trim()) {
  throw new ValidationError("Study session is invalid", [
    { field: "course", message: "Course is required" },
  ]);
}
~~~

The class lets an HTTP or CLI boundary choose a response without searching message text.

## 6. Reading a stack trace

A stack usually begins near the thrown error and lists calling frames. Start with the first frame belonging to your code.

~~~text
TypeError: Duration must be a positive integer
    at parseDuration (.../duration.js:6:11)
    at createSession (.../session.js:10:20)
    at main (.../index.js:4:19)
~~~

Read it as: main called createSession, which called parseDuration, which threw.

## 7. Using the debugger

Start with a breakpoint before application code:

~~~powershell
node --inspect-brk examples/06-debug.js
~~~

Attach your editor or browser developer tools. Then:

- Continue to a breakpoint
- Step over the current line
- Step into a function
- Inspect local variables
- Inspect the call stack

Use console output for quick evidence and the debugger when state changes are difficult to follow.

## 8. Last-resort process handlers

An unhandled rejection or uncaught exception means normal control flow failed. A server may log the failure and begin graceful shutdown, but it should not pretend that execution is definitely safe.

Do not use global handlers as a substitute for local error design.

## Real-world applications and edge cases

### Where this appears

- Validation errors become controlled 422 API responses.
- Database uniqueness errors become conflict responses.
- Downstream timeouts receive contextual causes and retry decisions.
- Fatal startup failures stop an unhealthy process before it receives traffic.

### Edge cases to investigate

- An error occurs after response headers were sent, so the server cannot replace the response with normal JSON.
- A catch block converts every failure into “not found,” hiding database outages as client mistakes.
- A retry wrapper catches a programmer error and repeats the same defect several times.
- An error message contains a database URL, token, query, or personal data and is returned to the client.
- Error serialization loses name, stack, code, and cause because Error properties are not ordinary enumerable fields.
- An unhandled rejection occurs after the original request has completed.
- A global uncaught-exception handler logs the event but continues using potentially corrupted process state.
- Several layers log the same failure, producing noisy duplicate incident evidence.

Decide which boundary can recover, which boundary translates the error, which boundary logs it, and which failures require the process to stop.

## Pause and predict

What happens here?

~~~javascript
async function fail() {
  throw new Error("No data");
}

try {
  fail();
  console.log("After call");
} catch (error) {
  console.log("Caught");
}
~~~

The catch does not receive the rejection because fail was not awaited. Add await and reconsider the output.

## Guided practice

Build validateSession(input) that returns a normalized study session or throws ValidationError with details for every invalid field.

Test:

- Valid course and duration
- Missing course
- Whitespace-only course
- String duration
- Zero duration
- Multiple invalid fields

At the CLI boundary, show validation details but keep unexpected stack traces for developers.

## Common mistakes

- Throwing strings
- Catching too early and losing context
- Catching and doing nothing
- Matching error.message text instead of using error types or codes
- Forgetting to await a rejecting promise
- Exposing stack traces to API clients
- Continuing normal operation after a fatal process-level failure

## Independent challenge

Create loadStudyData(path) that:

1. Reads a JSON file.
2. Adds context to file-system failures.
3. Distinguishes invalid JSON from a missing file.
4. Validates that the parsed result is an array.
5. Preserves original errors with cause.

Write a small caller that prints friendly messages but can still log technical details during development.

## Knowledge check

- Why throw Error objects?
- What does the cause option preserve?
- Where should expected errors normally be translated?
- How does a rejected promise escape a try block?
- Which stack frame should you inspect first?

## Explore with an AI agent

- Review my error boundaries and identify catches that are too early or too broad.
- Give me a broken async example and make me find why its rejection is unhandled.
- Act as a debugger: reveal one variable at a time while I diagnose a stack trace.
- Compare error classes, error codes, and result objects for expected failures.

## Official reading

- [Node.js errors](https://nodejs.org/api/errors.html)
- [Error cause in the Node.js errors guide](https://nodejs.org/api/errors.html#errorcause)
- [Node.js debugger](https://nodejs.org/api/debugger.html)
- [Process events](https://nodejs.org/api/process.html#process-event-uncaughtexception)

## Definition of done

- Errors are objects with useful messages.
- Async errors remain connected through await or return.
- Expected failures are handled at an appropriate boundary.
- You can read a stack trace and inspect code with a debugger.
