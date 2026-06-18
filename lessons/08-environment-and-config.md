# Lesson 08 - Environment Variables and Configuration

> Reading time: about 22 minutes  
> Practice time: about 40 minutes

## Outcome

Load configuration from the environment, convert and validate every value at startup, document required settings, and keep secrets out of source control.

## 1. Configuration belongs outside application logic

The same application may run locally, in tests, and in production with different ports, database addresses, and log levels.

Hard-coded environment details force code changes:

~~~javascript
const databaseUrl = "postgresql://admin:secret@production/db";
~~~

That is both inflexible and dangerous. Read deployment-specific values from the environment instead.

## 2. process.env values are strings

~~~javascript
console.log(process.env.PORT);
~~~

The result is a string or undefined. Even PORT=3000 produces "3000", not the number 3000.

### Convert explicitly

~~~javascript
const port = Number(process.env.PORT ?? 3000);

if (!Number.isInteger(port) || port < 1 || port > 65535) {
  throw new Error("PORT must be an integer from 1 through 65535");
}
~~~

The default applies only when the variable is null or undefined. Validation runs before the server begins accepting traffic.

## 3. Node.js can load .env files

Modern Node.js supports environment files:

~~~powershell
node --env-file=.env src/server.js
~~~

Example .env:

~~~dotenv
PORT=3000
LOG_LEVEL=debug
DATABASE_URL=postgresql://user:password@localhost:5432/study_tracker
~~~

Do not commit the real .env file. Commit .env.example containing names and safe examples.

## 4. Centralize configuration

Create src/config.js:

~~~javascript
const allowedLogLevels = new Set([
  "debug",
  "info",
  "warn",
  "error",
]);

function required(name) {
  const value = process.env[name];

  if (!value) {
    throw new Error(name + " is required");
  }

  return value;
}

function readPort() {
  const port = Number(process.env.PORT ?? 3000);

  if (!Number.isInteger(port) || port < 1 || port > 65535) {
    throw new Error("PORT must be an integer from 1 through 65535");
  }

  return port;
}

function readLogLevel() {
  const level = process.env.LOG_LEVEL ?? "info";

  if (!allowedLogLevels.has(level)) {
    throw new Error("LOG_LEVEL is not supported");
  }

  return level;
}

export const config = Object.freeze({
  port: readPort(),
  logLevel: readLogLevel(),
  databaseUrl: required("DATABASE_URL"),
});
~~~

The rest of the application receives typed, validated values instead of repeatedly reading process.env.

## 5. Required values and defaults

Use a default when one behavior is genuinely safe across environments:

~~~javascript
const logLevel = process.env.LOG_LEVEL ?? "info";
~~~

Require a value when guessing would connect to the wrong service, weaken security, or hide a deployment mistake:

~~~javascript
const tokenSecret = required("TOKEN_SECRET");
~~~

A production service that starts with accidental settings is often worse than one that refuses to start.

## 6. Secrets need special care

Secrets include passwords, private keys, tokens, and signing material.

- Do not commit them.
- Do not print them.
- Do not place them in error messages.
- Inject them through the deployment platform.
- Rotate them when exposure is suspected.
- Give them only the permissions they need.

.env is a local convenience, not a production secret manager.

## 7. Test configuration without changing global state everywhere

A pure parser is easier to test:

~~~javascript
export function parseConfig(environment) {
  const port = Number(environment.PORT ?? 3000);

  if (!Number.isInteger(port)) {
    throw new Error("PORT must be an integer");
  }

  return {
    port,
    logLevel: environment.LOG_LEVEL ?? "info",
  };
}

export const config = Object.freeze(parseConfig(process.env));
~~~

Tests can pass plain objects without mutating the real process environment.

## Real-world applications and edge cases

### Where this appears

- The same build runs in development, staging, and production with different settings.
- Containers receive secrets and service addresses from their platform.
- Tests pass controlled configuration objects.
- A deployment changes feature behavior without rebuilding application code.

### Edge cases to investigate

- A variable exists but contains an empty string, whitespace, or a misspelled enum value.
- Boolean parsing treats the nonempty string "false" as true.
- Two replicas start with different environment values during a partial rollout.
- A secret rotates while long-running processes still hold the old value in memory.
- A multiline certificate or private key is damaged by quoting or newline handling.
- Environment variable names behave differently across operating systems.
- Startup logs print a complete configuration object and expose credentials.
- A dynamic feature flag is treated as startup configuration, so changes require process restart.
- Test code mutates process.env and leaks state into another concurrently running test.

Separate immutable startup configuration from truly dynamic control data. Validate both, but give dynamic flags an explicit source, refresh policy, and failure behavior.

## Pause and predict

What values result from:

~~~javascript
process.env.FEATURE_ENABLED = "false";
console.log(Boolean(process.env.FEATURE_ENABLED));
~~~

The string "false" is nonempty, so Boolean returns true. Booleans require explicit parsing.

## Guided practice

Create:

~~~dotenv
# .env.example
PORT=3000
LOG_LEVEL=info
DATABASE_URL=postgresql://user:password@localhost:5432/study_tracker
TOKEN_SECRET=replace-me
~~~

Then write parseConfig that:

- Converts PORT to a number
- Accepts LOG_LEVEL from a fixed list
- Requires DATABASE_URL and TOKEN_SECRET
- Reports every invalid field in one startup error
- Returns a frozen object
- Never includes secret values in the error

## Common mistakes

- Assuming environment values already have number or boolean types
- Committing .env
- Printing the configuration object with secrets included
- Reading process.env across many modules
- Silently choosing production-sensitive defaults
- Validating only when a route first needs the setting
- Treating .env as a production secret manager

## Independent challenge

Add NODE_ENV and a boolean ENABLE_REQUEST_LOGGING setting. Accept only explicit true and false strings. Write table-driven tests for valid, missing, and invalid input objects.

Then answer: which settings should have defaults, and which should stop startup?

## Knowledge check

- What type are process.env values?
- Why validate at startup?
- What is the purpose of .env.example?
- Why centralize configuration?
- Why is Boolean("false") surprising?

## Explore with an AI agent

- Threat-model my configuration module for accidental secret exposure.
- Generate a table of valid and invalid environment inputs for my tests.
- Explain how my chosen deployment platform injects secrets, using its official documentation.
- Review which defaults are safe and which hide deployment mistakes.

## Official reading

- [Environment variables and .env files](https://nodejs.org/api/environment_variables.html)
- [Process environment](https://nodejs.org/api/process.html#processenv)
- [Node.js command-line options](https://nodejs.org/api/cli.html)
- [Node.js security best practices](https://nodejs.org/en/learn/getting-started/security-best-practices)

## Definition of done

- All application code receives validated configuration.
- Numeric and boolean values are explicitly parsed.
- .env is ignored and .env.example is documented.
- No secret appears in code, logs, or error text.
