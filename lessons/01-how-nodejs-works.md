# Lesson 01 - How Node.js Works

> Reading time: about 15 minutes  
> Practice time: about 25 minutes

## Outcome

By the end, you should be able to explain what Node.js is, what it adds to JavaScript, why its input/output model matters, and when you would choose it.

## Why this matters

Many later topics become confusing if Node.js is treated as a mysterious program that simply runs JavaScript. A better mental model is:

> JavaScript is the language. V8 executes the language. Node.js combines V8 with operating-system APIs and an event-driven runtime.

That sentence is the foundation of the course.

## 1. Node.js is a runtime, not a language

JavaScript defines language features such as variables, functions, objects, promises, and modules. JavaScript does not itself define how to read a file or open a network server.

Node.js supplies those capabilities through modules such as:

- node:fs for files
- node:http for HTTP servers and clients
- node:path for portable paths
- node:process for information about the current process

### Example: JavaScript plus a Node.js API

~~~javascript
import { platform } from "node:os";

const course = {
  name: "Node.js Learning Course",
  started: true,
};

console.log(course.name);
console.log("Operating system:", platform());
~~~

The object and console call are ordinary JavaScript. The import from node:os asks the Node.js runtime for an operating-system feature.

## 2. V8 executes JavaScript

V8 is the JavaScript engine also used by Chromium-based browsers. It parses and executes JavaScript. Node.js embeds V8, then adds a runtime around it.

This explains why the same language works in a browser and Node.js while some global APIs differ.

### Example: same language, different environment

This works in a browser:

~~~javascript
document.querySelector("h1");
~~~

It fails in Node.js because Node.js does not provide a Document Object Model.

This works in Node.js:

~~~javascript
console.log(process.version);
~~~

It normally fails in a browser because browsers do not expose the Node.js process object.

### Useful comparison

| Capability | Browser | Node.js |
|---|---:|---:|
| JavaScript language | Yes | Yes |
| document and window | Yes | No |
| File system access | Highly restricted | Yes |
| Create an HTTP server | No | Yes |
| process information | No | Yes |
| fetch HTTP data | Yes | Yes |

## 3. Node.js is designed around events and nonblocking I/O

Programs often spend time waiting: for a file, database, timer, or network response. Node.js usually starts that operation and continues handling other work instead of making the JavaScript thread wait idly.

### Example: starting file work

~~~javascript
import { readFile } from "node:fs";

console.log("1. Before reading");

readFile("notes.txt", "utf8", (error, text) => {
  if (error) {
    console.error("Could not read notes.txt");
    return;
  }

  console.log("3. File contents:", text);
});

console.log("2. After starting the read");
~~~

The likely output order is:

~~~text
1. Before reading
2. After starting the read
3. File contents: ...
~~~

The read begins, but its callback runs only after the operation completes. Node.js can perform other work while waiting.

### Important distinction

Asynchronous does not automatically mean that your JavaScript is running on many threads. Most application JavaScript runs on one main thread. Node.js coordinates operating-system facilities and an internal worker pool for certain operations. Later lessons cover the event loop and worker threads in detail.

## 4. Where Node.js fits well

Node.js is often a strong choice for:

- JSON APIs and web services
- Real-time network applications
- Command-line automation
- Build and developer tools
- Applications that coordinate many I/O operations
- Full-stack teams that want JavaScript across browser and server

It can still perform CPU-heavy work, but a long calculation on the main thread delays other requests. Worker threads, separate services, or another runtime may be better for sustained CPU-heavy workloads.

## Real-world applications and edge cases

### Where this appears

- An API server spends much of its time waiting for PostgreSQL or another service.
- A chat gateway coordinates thousands of mostly idle network connections.
- A command-line tool reads files and calls remote APIs.
- A build tool transforms many small source files.

### Edge cases to investigate

- One route performs a five-second synchronous calculation. Every unrelated request handled by that process waits.
- An application stores login sessions in memory. A restart or second instance makes some users appear logged out.
- A native dependency works on one operating system but has no compatible binary on the deployment platform.
- A serverless deployment repeatedly starts fresh processes, so initialization cost and process-local caches behave differently from a long-running server.
- The process accepts more work than its database can handle. Nonblocking I/O keeps accepting requests, but it does not create infinite downstream capacity.

When evaluating Node.js, ask what the program waits for, what consumes CPU, where state lives, and what happens when the process restarts.

## Pause and predict

Without running code, answer:

1. Is Node.js a programming language?
2. Who provides process: JavaScript or Node.js?
3. Why can file reading finish after later console output?
4. Would a video-encoding loop be naturally suited to the main Node.js thread?

## Guided practice

Create examples/01-runtime.js:

~~~javascript
import { cpus, homedir, platform } from "node:os";

console.log({
  nodeVersion: process.version,
  platform: platform(),
  homeDirectory: homedir(),
  logicalCpuCount: cpus().length,
});
~~~

Run:

~~~powershell
node examples/01-runtime.js
~~~

Now change one field, run it again, and explain which values came from JavaScript and which came from Node.js APIs.

## Common mistakes

### Calling Node.js a framework

Express is a framework. Node.js is the runtime underneath it.

### Assuming nonblocking means instant

An asynchronous operation can still be slow. Nonblocking means the program can make progress elsewhere while waiting.

### Assuming Node.js can never use multiple threads

The main JavaScript execution model is usually single-threaded, but Node.js also uses internal threads and offers worker_threads for application code.

## Independent challenge

Write a short technology recommendation for four imaginary products:

1. A REST API for a mobile application
2. A command-line file renamer
3. A large scientific simulation
4. A real-time chat server

For each product, state whether Node.js is a sensible choice and explain why in two or three sentences.

## Knowledge check

- Define runtime in your own words.
- What relationship exists between Node.js and V8?
- Name three APIs supplied by Node.js.
- What does nonblocking I/O allow the process to do?
- Why can CPU-heavy code hurt a server?

## Explore with an AI agent

- Explain the boundary between V8, Node.js, libuv, and the operating system using one concrete file-reading example.
- Quiz me with five scenarios where I must choose between browser APIs and Node.js APIs.
- Give me a small experiment that demonstrates blocking versus nonblocking work without using third-party packages.

## Official reading

- [Introduction to Node.js](https://nodejs.org/en/learn/getting-started/introduction-to-nodejs)
- [Node.js documentation overview](https://nodejs.org/api/documentation.html)
- [Global objects](https://nodejs.org/api/globals.html)
- [Operating system module](https://nodejs.org/api/os.html)

## Definition of done

- You can explain Node.js without calling it a language or framework.
- You can distinguish JavaScript features from Node.js APIs.
- You can explain why Node.js is effective for I/O-heavy programs.
- You completed the technology recommendation in your own words.
