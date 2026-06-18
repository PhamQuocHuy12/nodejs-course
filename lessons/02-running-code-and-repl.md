# Lesson 02 - Running Code and Using the REPL

> Reading time: about 18 minutes  
> Practice time: about 30 minutes

## Outcome

Run Node.js code in several ways, read command-line input, communicate through standard streams, and report success or failure correctly.

## 1. Running a JavaScript file

The most common form is:

~~~powershell
node path/to/file.js
~~~

Node.js loads the entry file, evaluates its imports, runs top-level code, and keeps the process alive while active work remains.

### Example

Create examples/02-hello.js:

~~~javascript
const name = "Node learner";
console.log("Hello, " + name + "!");
~~~

Run:

~~~powershell
node examples/02-hello.js
~~~

When no timers, servers, open streams, or other active work remain, the process exits.

## 2. The REPL

REPL means Read, Evaluate, Print, Loop. Start it by running node without a file:

~~~powershell
node
~~~

Try:

~~~javascript
2 + 2
const minutes = [25, 40, 15]
minutes.reduce((total, value) => total + value, 0)
~~~

The REPL is useful for tiny experiments. It is not where a real application should live because its history is not a maintainable program.

Exit with:

~~~text
.exit
~~~

or press Ctrl+C twice.

## 3. The process object

The global process object represents the running Node.js process.

### Example: inspect the runtime

~~~javascript
console.log("Node version:", process.version);
console.log("Process ID:", process.pid);
console.log("Platform:", process.platform);
console.log("Working directory:", process.cwd());
~~~

The working directory is where the command was started. It is not necessarily the directory containing the JavaScript file.

## 4. Command-line arguments

Arguments are available in process.argv:

~~~javascript
console.log(process.argv);
~~~

The first entries are normally the Node.js executable and entry script. Application arguments begin at index 2.

### Example: greet a user

~~~javascript
const [name = "stranger"] = process.argv.slice(2);
console.log("Hello, " + name + "!");
~~~

Run:

~~~powershell
node examples/02-greet.js Linh
~~~

Destructuring the sliced array makes the application arguments easier to understand.

## 5. Standard output and standard error

Command-line programs have three standard streams:

- stdin receives input
- stdout carries normal output
- stderr carries warnings and errors

console.log writes normal output. console.error writes error output.

### Example: separate success and failure

~~~javascript
const [value] = process.argv.slice(2);
const number = Number(value);

if (!Number.isFinite(number)) {
  console.error("Usage: node square.js <number>");
  process.exitCode = 1;
} else {
  console.log(number ** 2);
}
~~~

This distinction matters because another program can capture normal output while still showing errors.

## 6. Exit status

By convention, exit code 0 means success. A nonzero value means failure.

Prefer assigning process.exitCode when the process can finish naturally:

~~~javascript
process.exitCode = 1;
~~~

Calling process.exit() stops the process quickly and can cut off pending output or cleanup.

## Real-world applications and edge cases

### Where this appears

- Database migration and seed commands
- Scheduled cleanup jobs
- Continuous-integration checks
- File-conversion utilities
- Administrative scripts used during incidents

### Edge cases to investigate

- A scheduler starts the script from a different working directory than your terminal.
- A filename contains spaces, Unicode, or characters interpreted by the shell.
- A child command relies on exit status, but the script prints an error and still exits with zero.
- Ctrl+C arrives while a file is being written or a database transaction is open.
- A CLI receives thousands of arguments and exceeds operating-system command-line limits.
- Normal output is piped into another program, so one accidental log line corrupts the data stream.
- A script is started twice by overlapping schedules and performs the same destructive action twice.

Real CLI tools need clear usage text, stable output, deliberate exit codes, signal handling, and idempotent behavior.

## Pause and predict

Given:

~~~powershell
node examples/tool.js red blue
~~~

What do you expect from:

~~~javascript
console.log(process.argv.slice(2));
console.log(process.argv[2]);
console.log(process.cwd());
~~~

Which answer depends on the directory where the command was started?

## Guided practice: age calculator

Create examples/02-age.js:

~~~javascript
const [name, birthYearText] = process.argv.slice(2);
const birthYear = Number(birthYearText);
const currentYear = new Date().getFullYear();

if (!name || !Number.isInteger(birthYear)) {
  console.error("Usage: node examples/02-age.js <name> <birth-year>");
  process.exitCode = 1;
} else {
  const approximateAge = currentYear - birthYear;
  console.log(name + " is approximately " + approximateAge + " years old.");
}
~~~

Test at least these cases:

~~~powershell
node examples/02-age.js Mai 2000
node examples/02-age.js Mai
node examples/02-age.js Mai unknown
~~~

Notice that Number("") becomes 0. Validation deserves deliberate thought even in tiny scripts.

## Common mistakes

- Reading process.argv[0] as the first application argument
- Using process.exit(1) before buffered output or cleanup can finish
- Sending error messages to stdout
- Assuming process.cwd() is the script directory
- Treating any value returned by Number as valid

## Independent challenge

Build examples/02-study-total.js. It should accept any number of study-duration arguments:

~~~powershell
node examples/02-study-total.js 25 40 15
~~~

Expected output:

~~~text
Sessions: 3
Total minutes: 80
Average minutes: 26.67
~~~

Reject missing, nonnumeric, zero, and negative durations. Set a nonzero exit code without calling process.exit().

## Knowledge check

- When should you use a file instead of the REPL?
- What are the first two process.argv entries?
- Why keep stdout and stderr separate?
- What does a nonzero exit status communicate?
- How can process.exit() lose work?

## Explore with an AI agent

- Show me how PowerShell redirects stdout and stderr separately, then explain why CLI tools care.
- Review my argument validation without rewriting the whole program.
- Give me three edge cases for my study-total script that I probably missed.

## Official reading

- [Running Node.js scripts from the command line](https://nodejs.org/en/learn/command-line/run-nodejs-scripts-from-the-command-line)
- [REPL API](https://nodejs.org/api/repl.html)
- [Process API](https://nodejs.org/api/process.html)
- [Command-line options](https://nodejs.org/api/cli.html)

## Definition of done

- You can run files and use the REPL.
- You can read and validate application arguments.
- Normal results and errors use different streams.
- Failure cases produce a nonzero exit status.
