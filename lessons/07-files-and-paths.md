# Lesson 07 - Files, Paths, and Directories

> Reading time: about 25 minutes  
> Practice time: about 50 minutes

## Outcome

Build portable paths, read and write files with promise-based APIs, parse JSON safely, and understand why simple file storage has concurrency limits.

## 1. Prefer promise-based file APIs for application flow

Node.js exposes file-system APIs in callback, synchronous, and promise forms.

~~~javascript
import { readFile } from "node:fs/promises";

const text = await readFile("notes.txt", "utf8");
console.log(text);
~~~

The encoding matters. Without utf8, readFile resolves to a Buffer containing bytes rather than a string.

Synchronous operations are convenient for tiny startup scripts but block the main thread while they run. Server request paths should normally use asynchronous APIs.

## 2. File operations fail for normal reasons

~~~javascript
try {
  const text = await readFile("notes.txt", "utf8");
  console.log(text);
} catch (error) {
  if (error.code === "ENOENT") {
    console.error("notes.txt does not exist");
  } else {
    throw error;
  }
}
~~~

ENOENT is a machine-readable Node.js error code. Do not identify error categories by matching human message text.

Other failures include permission problems, invalid paths, locked resources, and malformed data after reading.

## 3. Build portable paths

Windows and POSIX systems represent paths differently. node:path handles separators and normalization.

~~~javascript
import path from "node:path";

const dataPath = path.join("data", "sessions.json");
console.log(dataPath);
~~~

Do not manually join paths with a slash or backslash.

Useful operations:

~~~javascript
path.basename("/courses/node.md");
path.extname("/courses/node.md");
path.dirname("/courses/node.md");
path.resolve("data", "sessions.json");
~~~

## 4. Working directory versus module location

process.cwd() returns the directory where the command was started.

~~~javascript
console.log(process.cwd());
~~~

An ESM module can locate a resource relative to itself with import.meta.url:

~~~javascript
const dataUrl = new URL("../data/sessions.json", import.meta.url);
const text = await readFile(dataUrl, "utf8");
~~~

Use the working directory for user-supplied project paths. Use module-relative URLs for files shipped beside your code.

## 5. Reading JSON

JSON reading has two separate failure stages:

~~~javascript
import { readFile } from "node:fs/promises";

async function readJson(file) {
  const text = await readFile(file, "utf8");

  try {
    return JSON.parse(text);
  } catch (error) {
    throw new SyntaxError("File contains invalid JSON", {
      cause: error,
    });
  }
}
~~~

Reading can fail because the file is unavailable. Parsing can fail because its contents are not valid JSON. Keeping those stages separate improves messages.

## 6. Writing JSON

~~~javascript
import { writeFile } from "node:fs/promises";

const sessions = [
  { course: "Node.js", minutes: 45 },
];

await writeFile(
  "sessions.json",
  JSON.stringify(sessions, null, 2) + "\n",
  "utf8",
);
~~~

writeFile replaces the destination by default. JSON.stringify with indentation makes a learning file readable.

## 7. Creating directories

~~~javascript
import { mkdir } from "node:fs/promises";

await mkdir("data", { recursive: true });
~~~

The recursive option succeeds when parent directories need creation and does not fail merely because the target already exists.

## 8. Why a JSON file is not yet a database

Consider two requests:

1. Both read the same sessions array.
2. Each adds a different session.
3. Both overwrite the file.

The later write can erase the earlier change. A file also lacks queries, transactions, indexes, and robust multi-process coordination.

File storage is excellent for learning and small single-process tools. Lesson 11 introduces a real database.

## Real-world applications and edge cases

### Where this appears

- Configuration and migration files shipped with an application
- Temporary files during uploads or report generation
- Command-line import and export tools
- Log rotation and archival jobs
- Static assets and templates

### Edge cases to investigate

- A user-controlled filename escapes the intended directory through path traversal.
- A symbolic link changes what a seemingly safe path points to.
- The process crashes halfway through writeFile, leaving a partial or empty destination.
- Two requests perform read-modify-write and one update overwrites the other.
- A file is replaced between checking access and opening it, creating a time-of-check to time-of-use race.
- Text uses an unexpected encoding or begins with a byte-order marker.
- A large file is buffered entirely and exhausts memory instead of being streamed.
- Windows temporarily prevents rename or deletion because another process still holds the file.
- Temporary files accumulate after cancellations and eventually fill the disk.

Treat paths as untrusted input, write through controlled directories, prefer atomic replacement where possible, and always plan cleanup for failure paths.

## Pause and predict

If the program starts from C:\work but its module lives under C:\app\src, which location does process.cwd() describe? Which location anchors a URL created from import.meta.url?

## Guided practice: file-backed study log

Create functions:

- ensureDataDirectory()
- loadSessions()
- saveSessions(sessions)
- addSession(session)
- totalMinutes(sessions)

Requirements:

- Missing data file produces an empty array.
- Invalid JSON is reported, not silently replaced.
- Every session is validated.
- The output file ends with a newline.
- All normal file work uses node:fs/promises.

## Common mistakes

- Forgetting the text encoding
- Manually concatenating path separators
- Confusing process.cwd() with the module directory
- Catching every error and treating it as a missing file
- Reading, changing, and writing shared data without considering races
- Calling access before open and assuming the file cannot change between calls
- Using synchronous file APIs in server request handlers

## Independent challenge

Create an import script that:

1. Accepts an input JSON path from process.argv.
2. Resolves it from the working directory.
3. Validates every record.
4. Writes normalized records under data/sessions.json.
5. Reports how many records succeeded and failed.
6. Never overwrites existing valid data when the input is malformed.

Explain how you would make the final write safer using a temporary file and rename.

## Knowledge check

- Why specify utf8?
- When should a path be working-directory-relative?
- How do you recognize a missing-file error?
- Why separate reading from JSON parsing?
- What race can a read-modify-write sequence create?

## Explore with an AI agent

- Review my file paths for Windows and POSIX portability.
- Show me a safe temporary-file-then-rename write strategy and explain its limitations.
- Generate malformed file and JSON cases for my loader without writing the solution.
- Explain Buffers using the text encoding example from this lesson.

## Official reading

- [File system API](https://nodejs.org/api/fs.html)
- [Promise-based file system API](https://nodejs.org/api/fs.html#promises-api)
- [Path API](https://nodejs.org/api/path.html)
- [URL API and file URLs](https://nodejs.org/api/url.html)
- [Working directory with process.cwd](https://nodejs.org/api/process.html#processcwd)

## Definition of done

- Paths work regardless of platform separator.
- Expected file errors and parse errors are distinguishable.
- JSON writes are readable and deliberate.
- You can explain the storage design's concurrency limitation.
