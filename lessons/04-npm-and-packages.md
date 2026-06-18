# Lesson 04 - npm and Package Management

> Reading time: about 22 minutes  
> Practice time: about 40 minutes

## Outcome

Create a package, install dependencies deliberately, understand the lockfile, write repeatable scripts, and interpret semantic versions.

## 1. A package is a project boundary

In Node.js, a package is a directory described by package.json. The file records identity, module type, scripts, dependencies, and other metadata.

### Minimal course package

~~~json
{
  "name": "nodejs-learning-course",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "test": "node --test"
  }
}
~~~

Scripts give the team one stable command even if the underlying command later changes.

## 2. Creating package.json

Interactive initialization:

~~~powershell
npm init
~~~

Fast initialization with defaults:

~~~powershell
npm init -y
~~~

Read the result. Generated does not mean correct. Update the name, private flag, type, and scripts intentionally.

## 3. Dependencies and development dependencies

A runtime dependency is required when the application runs:

~~~powershell
npm install picocolors
~~~

A development dependency supports building, checking, or testing:

~~~powershell
npm install --save-dev eslint
~~~

They appear in different package.json sections:

~~~json
{
  "dependencies": {
    "picocolors": "^1.1.1"
  },
  "devDependencies": {
    "eslint": "^9.0.0"
  }
}
~~~

Version numbers here are examples. Your installed versions may differ.

## 4. Using an installed package

~~~javascript
import pc from "picocolors";

console.log(pc.green("Study session saved."));
console.error(pc.red("Study session failed."));
~~~

Node.js resolves the bare specifier picocolors through the package system. A relative import would begin with ./ or ../.

## 5. Semantic versioning

Semantic versions look like major.minor.patch:

- Patch: compatible bug fix
- Minor: backward-compatible feature
- Major: potentially incompatible change

For version 3.4.2:

- 3 is major
- 4 is minor
- 2 is patch

A range beginning with a caret usually allows compatible updates within the current major version when the major is nonzero. Always let npm calculate ranges instead of guessing obscure edge cases.

## 6. The lockfile

package.json may allow a range. package-lock.json records the concrete dependency tree selected for this project.

Commit package-lock.json for an application. It improves repeatability and lets npm ci reproduce the locked tree in automation.

Use:

~~~powershell
npm install
~~~

while intentionally changing dependencies.

Use:

~~~powershell
npm ci
~~~

for a clean, lockfile-based installation in continuous integration or a fresh application checkout.

Do not commit node_modules. It is generated, large, and platform-dependent.

## 7. npm scripts

Add:

~~~json
{
  "scripts": {
    "start": "node src/index.js",
    "dev": "node --watch src/index.js",
    "test": "node --test",
    "check": "npm test"
  }
}
~~~

Run:

~~~powershell
npm run dev
npm test
npm run check
~~~

Some conventional scripts, including test and start, can omit the word run.

## 8. Local execution and npx

Packages are normally installed locally to the project. npm scripts automatically find their local executables.

npx can run a package executable without a global installation:

~~~powershell
npx <package-command>
~~~

Before running an unfamiliar command through npx, verify the package name and publisher. Typos can execute the wrong package.

## Pause and predict

If package.json contains:

~~~json
{
  "scripts": {
    "learn": "node examples/lesson.js"
  }
}
~~~

Which command runs it? Which file should be committed: package-lock.json, node_modules, both, or neither?

## Guided practice

1. Initialize a package.
2. Set private to true and type to module.
3. Add start, dev, test, and check scripts.
4. Install one small runtime dependency.
5. Import and use it from src/index.js.
6. Inspect package.json and package-lock.json.
7. Remove node_modules and restore it with npm ci.

Explain every changed file before continuing.

## Choosing dependencies

Before adding a package, ask:

- Does Node.js already provide the feature?
- Is the package maintained?
- Is its license suitable?
- How large is its dependency tree?
- Does its documentation match the current release?
- Would a small local function be clearer?

Fewer dependencies are not automatically better, but every dependency is code you operate without writing.

## Real-world applications and edge cases

### Where this appears

- A production deployment reproduces the dependency tree with npm ci.
- A security update changes one transitive dependency.
- A monorepo shares packages through npm workspaces.
- A library supports several Node.js versions and publishes semantic releases.

### Edge cases to investigate

- package.json and package-lock.json disagree after a difficult merge, causing npm ci to fail.
- A package runs an installation script with the same permissions as the developer or build agent.
- A patch release introduces a regression even though semantic versioning says it should be compatible.
- A development dependency is imported by production code but omitted from the production installation.
- An optional native dependency installs on one platform and fails on another.
- A package is renamed or compromised and a look-alike package uses a common typo.
- A transitive dependency is vulnerable, but upgrading it requires a new major version of a direct dependency.
- A package on major version zero may use compatibility conventions differently from a mature stable package.

Dependency review is operational work: inspect lockfile changes, minimize install privileges, automate vulnerability awareness, and test updates before deployment.

## Common mistakes

- Installing everything globally
- Deleting or ignoring package-lock.json in an application
- Editing node_modules directly
- Confusing dependencies with devDependencies
- Blindly running npx commands copied from strangers
- Treating a major upgrade as automatically safe
- Adding a package before checking built-in Node.js APIs

## Independent challenge

Create a tiny command-line package called study-summary:

- It accepts durations from process.argv.
- It prints the total using one small formatting dependency.
- It has start, dev, and test scripts.
- It rejects invalid durations.
- A fresh checkout can use npm ci and npm test.

Write three sentences explaining why your dependency belongs in dependencies rather than devDependencies.

## Knowledge check

- What role does package.json play?
- How does package-lock.json differ?
- When should npm ci be preferred?
- What does a major version increase communicate?
- Why prefer local package installation?

## Explore with an AI agent

- Inspect my package.json and explain every field without changing it.
- Compare npm install and npm ci for an application repository.
- Help me evaluate whether a package is necessary using Node.js built-in alternatives.
- Give me five semantic-version ranges and quiz me on the updates they permit.

## Official reading

- [About npm](https://docs.npmjs.com/about-npm)
- [package.json documentation](https://docs.npmjs.com/cli/latest/configuring-npm/package-json)
- [Semantic versioning with npm](https://docs.npmjs.com/about-semantic-versioning)
- [npm install](https://docs.npmjs.com/cli/latest/commands/npm-install)
- [npm ci](https://docs.npmjs.com/cli/latest/commands/npm-ci)
- [Node.js package documentation](https://nodejs.org/api/packages.html)

## Definition of done

- package.json accurately describes your project.
- Runtime and development dependencies are categorized correctly.
- package-lock.json is committed and node_modules is ignored.
- One command can run each common project task.
