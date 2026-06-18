# Lesson 03 - Modules: CommonJS and ESM

> Reading time: about 22 minutes  
> Practice time: about 40 minutes

## Outcome

Split a program into focused modules, use ECMAScript modules confidently, and recognize CommonJS when reading older Node.js code.

## Why modules exist

A module is a file with its own scope and an explicit public interface. Modules help answer two questions:

1. Which details belong together?
2. Which details may other parts of the program use?

Without boundaries, every file can depend on every implementation detail. The program becomes a bowl of spaghetti that has somehow learned to deploy.

## 1. ECMAScript modules

ECMAScript modules, usually called ESM, use export and import.

### Export a named value

Create examples/03-math.js:

~~~javascript
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}
~~~

Create examples/03-use-math.js:

~~~javascript
import { add, subtract } from "./03-math.js";

console.log(add(4, 3));
console.log(subtract(4, 3));
~~~

Relative ESM imports include the file extension. The braces select named exports.

## 2. Enabling ESM

One common approach is a package.json file:

~~~json
{
  "name": "nodejs-learning-course",
  "private": true,
  "type": "module"
}
~~~

Within that package scope, .js files are interpreted as ESM. A .mjs file is always ESM. A .cjs file is always CommonJS.

The private field prevents accidental publication to the npm registry.

## 3. Named and default exports

A module can have several named exports and at most one default export.

~~~javascript
export const defaultDuration = 25;

export function formatMinutes(minutes) {
  return minutes + " min";
}

export default function createSession(course, minutes = defaultDuration) {
  return { course, minutes };
}
~~~

Import them:

~~~javascript
import createSession, {
  defaultDuration,
  formatMinutes,
} from "./session.js";
~~~

Named exports make refactoring and editor discovery clear. This course prefers named exports unless one value is unmistakably the module's main purpose.

## 4. Built-in modules

Use the node: prefix for Node.js built-ins:

~~~javascript
import { readFile } from "node:fs/promises";
import path from "node:path";
~~~

The prefix makes it obvious that these are built into Node.js rather than installed from npm.

## 5. CommonJS

Older Node.js code and many packages use CommonJS:

~~~javascript
function add(a, b) {
  return a + b;
}

module.exports = { add };
~~~

Import it with:

~~~javascript
const { add } = require("./math.cjs");
~~~

Do not mechanically mix require and import. Interoperability rules exist, but a beginner project is clearer when it chooses one system. We use ESM.

## 6. Module scope

A top-level variable in a module is not automatically global:

~~~javascript
const secret = "only this module can see me";

export function revealLength() {
  return secret.length;
}
~~~

Other modules can call revealLength but cannot import secret. This is encapsulation: expose behavior while keeping internal details private.

## 7. Side effects

Importing a module evaluates its top-level code once:

~~~javascript
console.log("Database module loaded");
~~~

A module that opens a server or database during import is difficult to test. Prefer exported setup functions:

~~~javascript
export function connectDatabase() {
  // Connect only when the application asks.
}
~~~

## Real-world applications and edge cases

### Where this appears

- Controllers import services while services import repositories.
- Shared validation and error modules define application-wide contracts.
- A package exposes a small public API while hiding internal helpers.
- Tests replace repository implementations through dependency injection.

### Edge cases to investigate

- Module A imports Module B while Module B imports Module A. One side may observe partially initialized exports.
- An imported configuration module validates environment variables immediately, making unrelated tests fail during import.
- A module opens a database connection as a side effect, so importing it keeps the test process alive.
- Code works on a case-insensitive Windows file system but fails on Linux because the import uses the wrong filename case.
- A dependency ships only ESM while older code expects require, or the reverse.
- Two copies of the same package are installed, so identity checks and shared singleton assumptions fail.
- A module's public interface exposes database entities, making later persistence changes spread through the application.

Treat the import graph as architecture: dependencies should generally point inward toward stable business rules, not form a web of circular side effects.

## Pause and predict

Suppose session.js exports:

~~~javascript
export const minutes = 25;
export default function start() {}
~~~

Which import is valid?

~~~javascript
import start, { minutes } from "./session.js";
~~~

or:

~~~javascript
import { start, minutes } from "./session.js";
~~~

Explain why.

## Guided practice: calculator modules

Create:

~~~text
examples/
  calculator/
    operations.js
    display.js
    index.js
~~~

operations.js exports add, subtract, multiply, and divide. Division by zero throws an Error.

display.js exports a function that formats output such as:

~~~text
12 / 3 = 4
~~~

index.js imports both modules and demonstrates every operation.

Keep argument validation in operations.js so every caller receives the same rules.

## Common mistakes

- Forgetting the relative path prefix such as ./
- Omitting the .js extension from relative ESM imports
- Using module.exports inside an ESM .js file
- Exporting every internal helper
- Starting servers or opening database connections as import side effects
- Creating circular imports between modules

## Independent challenge

Create a study-session module with:

- createSession(course, minutes)
- calculateTotal(sessions)
- calculateAverage(sessions)
- formatSession(session)

Reject empty course names and durations that are not positive integers. Write an index file that imports the public functions and demonstrates valid and invalid data.

## Knowledge check

- What problem does module scope solve?
- How does package.json select ESM for .js files?
- What is the difference between named and default exports?
- Why use the node: prefix?
- Why can import-time side effects make tests difficult?

## Explore with an AI agent

- Compare CommonJS loading and ESM loading without assuming one is universally better.
- Review the public interface of my study-session module. Which details should remain private?
- Draw the dependency direction between my index, display, and operations modules.

## Official reading

- [ECMAScript modules](https://nodejs.org/api/esm.html)
- [CommonJS modules](https://nodejs.org/api/modules.html)
- [Packages and the type field](https://nodejs.org/api/packages.html)
- [Node.js built-in modules](https://nodejs.org/api/modules.html#built-in-modules)

## Definition of done

- Your project runs as ESM.
- Relative imports include extensions.
- Module interfaces are smaller than their implementations.
- You can recognize and explain equivalent CommonJS syntax.
