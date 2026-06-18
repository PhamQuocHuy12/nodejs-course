# Lesson 16 - Streams and Large Data

> Reading time: about 32 minutes  
> Practice time: about 75 minutes

## Outcome

Process data incrementally with readable, writable, and transform streams; use pipeline for reliable composition; and explain how backpressure protects memory.

## Why streams exist

readFile loads an entire file before you can use it:

~~~javascript
const data = await readFile("large-video.mp4");
~~~

That is simple and correct for small files. For a multi-gigabyte file, memory grows with file size.

A stream handles a sequence of chunks:

~~~javascript
const stream = createReadStream("large-video.mp4");
~~~

The program can begin processing before the full file is available.

## 1. Four stream categories

- Readable: produces data
- Writable: consumes data
- Duplex: both readable and writable
- Transform: duplex stream whose output is derived from input

Examples:

- File read stream: readable
- HTTP response: writable from server code
- TCP socket: duplex
- Compression stream: transform

## 2. Read chunks with async iteration

~~~javascript
import { createReadStream } from "node:fs";

const stream = createReadStream("notes.txt", {
  encoding: "utf8",
});

for await (const chunk of stream) {
  console.log("Received characters:", chunk.length);
}
~~~

Chunk boundaries are implementation details. One line or JSON object can be split across chunks, and one chunk can contain several logical records.

## 3. Writable streams and backpressure

write returns a boolean:

~~~javascript
const canContinue = writable.write(chunk);
~~~

false means the writable buffer has reached its threshold. The producer should wait for the drain event before writing more.

Backpressure is a flow-control signal. Without it, a fast producer can queue data faster than a slow consumer can process it, causing memory growth.

## 4. Use pipeline for composition

~~~javascript
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
import { createGzip } from "node:zlib";

await pipeline(
  createReadStream("sessions.json"),
  createGzip(),
  createWriteStream("sessions.json.gz"),
);
~~~

pipeline:

- Connects stream stages
- Propagates errors
- Resolves when the full operation completes
- Destroys connected streams when appropriate after failure

Prefer it over manually attaching several pipe and error handlers.

## 5. Create a transform stream

~~~javascript
import { Transform } from "node:stream";

const uppercase = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  },
});
~~~

A transform receives chunks and emits changed chunks. callback reports either an error or output.

For text protocols, remember that a character can span byte chunks. Set an encoding or use a decoder when byte boundaries matter.

## 6. Object mode

Normal streams move strings, Buffers, or typed arrays. Object mode allows arbitrary JavaScript values:

~~~javascript
import { Readable } from "node:stream";

const sessions = Readable.from([
  { course: "Node.js", minutes: 25 },
  { course: "PostgreSQL", minutes: 40 },
]);

for await (const session of sessions) {
  console.log(session.course);
}
~~~

Object mode changes how buffering thresholds are counted: objects rather than bytes.

## 7. Stream an HTTP response

~~~javascript
import { createReadStream } from "node:fs";
import { pipeline } from "node:stream/promises";

app.get("/exports/sessions", async (request, response, next) => {
  try {
    response.set({
      "content-type": "text/csv; charset=utf-8",
      "content-disposition": "attachment; filename=sessions.csv",
    });

    await pipeline(
      createReadStream("data/sessions.csv"),
      response,
    );
  } catch (error) {
    if (!response.headersSent) {
      next(error);
    }
  }
});
~~~

After headers or body bytes are sent, a normal JSON error may no longer be possible. Streaming endpoints need careful error and disconnect behavior.

## 8. Generate CSV incrementally

CSV needs escaping. A field containing a comma, quote, or newline should be quoted, and embedded quotes should be doubled.

~~~javascript
function escapeCsv(value) {
  const text = String(value);

  if (/[",\r\n]/.test(text)) {
    return '"' + text.replaceAll('"', '""') + '"';
  }

  return text;
}
~~~

A TypeORM query should fetch rows in bounded pages or use a database cursor. Calling find on every row and then streaming only the formatting does not solve database memory usage.

## 9. Cancellation and cleanup

Clients can disconnect before export finishes. Observe request or response closure and stop database or file work where possible.

AbortSignal is the standard cancellation representation in many Node.js APIs:

~~~javascript
const controller = new AbortController();
const { signal } = controller;
~~~

Cancellation should release file handles, database cursors, and other resources.

## Real-world applications and edge cases

### Where this appears

- Browser uploads pass through an API to object storage.
- CSV and JSON-lines exports contain millions of records.
- Proxies forward HTTP bodies without buffering them completely.
- Compression and encryption transform data in motion.
- Log processors consume continuous event streams.

### Edge cases to investigate

- A UTF-8 character or line ending is divided across chunk boundaries.
- A CSV field contains quotes, commas, carriage returns, or spreadsheet formulas.
- The database cursor produces rows faster than the network client consumes them.
- A client disconnects after the server has performed expensive work but before the download finishes.
- An error occurs after response headers and several chunks were sent.
- A compressed upload expands into enormous output, creating a decompression bomb.
- A partial destination file remains after cancellation and is mistaken for a complete export.
- Several pipeline stages each buffer data, making total memory higher than expected.
- A checksum must describe the complete stream, but failure occurs near the end.
- Object-mode transforms accidentally retain references to every processed object.

Streaming designs need an explicit record-framing strategy, cancellation path, partial-output policy, and end-to-end backpressure.

## Pause and predict

- Can one data event be assumed to contain one line?
- What does writable.write returning false mean?
- Does streaming formatting help if all database rows were already loaded?
- Can a server always send JSON after stream bytes were sent?

## Guided practice

Create a large text file and compare:

1. readFile plus one large in-memory transformation
2. createReadStream plus a Transform and pipeline

Record:

- File size
- Duration
- Peak heap usage
- Complexity of error handling

Then add a streaming CSV export to the Study Tracker API.

## Common mistakes

- Assuming chunk boundaries match records
- Ignoring backpressure
- Handling only the readable stream's errors
- Loading all rows before starting a streamed response
- Writing invalid CSV escaping
- Ignoring client disconnects
- Trying to send a JSON error after headers are committed
- Using streams for tiny data where a buffer is clearer

## Independent challenge

Build a streaming import:

1. Read a large line-oriented file.
2. Parse records incrementally.
3. Validate each record.
4. Insert in bounded batches inside transactions.
5. Report accepted and rejected counts.
6. Stop cleanly on cancellation.

Define whether one invalid row aborts everything or produces a partial import. That is a product decision, not merely a stream decision.

## Knowledge check

- What are readable, writable, duplex, and transform streams?
- What problem does backpressure solve?
- Why prefer pipeline?
- How does object mode differ?
- Which resources must cancellation release?

## Explore with an AI agent

- Trace backpressure through my exact stream pipeline.
- Give me chunk-boundary edge cases for my line parser.
- Review whether my CSV endpoint truly uses bounded memory.
- Explain how AbortSignal travels from client disconnect to database cancellation.

## Official reading

- [Node.js streams API](https://nodejs.org/api/stream.html)
- [Backpressuring in streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
- [File streams](https://nodejs.org/api/fs.html#filehandlecreatereadstreamoptions)
- [Zlib compression](https://nodejs.org/api/zlib.html)

## Definition of done

- Large input is processed with bounded memory.
- pipeline owns stream completion and errors.
- CSV output is correctly escaped.
- Disconnect and failure paths release resources.
