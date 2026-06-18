# Node.js Learning Course

A 24-lesson, self-study Node.js course built from official documentation and primary project sources. It is designed to be comfortable to read on a phone and useful even when no terminal is nearby.

The course moves from runtime fundamentals to building, testing, securing, and operating a production-style API. Theory and practice stay together: every important idea is followed by code, an explanation of the code, and something for you to try.

## How the course works

For each lesson:

1. Read the chapter once on your phone. Do not memorize it.
2. Answer the short **Pause and predict** questions before revealing the result.
3. When you are at a computer, type the examples yourself.
4. Complete **Guided practice**, then attempt the independent challenge.
5. Use the supplied **Explore with an AI agent** prompts to go deeper.
6. Ask the agent for hints before requesting a complete solution.
7. Request a code review and mark the lesson in [PROGRESS.md](PROGRESS.md).

Use this prompt to begin:

> Teach me Lesson 01 from this repository. Use the lesson as the source of truth. Explain one concept at a time, check my understanding with short questions, and give hints before solutions.

## Chapter design

Each lesson contains:

- A practical outcome and estimated reading time
- Theory in short sections
- Runnable examples beside the theory
- An explanation of what each example demonstrates
- Real-world applications and topic-specific edge cases
- Common mistakes and debugging advice
- Recall questions and an independent challenge
- Prompts for exploring the topic with an AI agent
- Links to official documentation

Official links are intentionally included in every chapter. Ask an AI agent to open those sources when you need current version-specific details.

## Course project

Small exercises begin immediately. Starting in Lesson 09, they become one **Study Tracker API** that eventually supports:

- Courses and study sessions
- PostgreSQL persistence
- Authentication and authorization
- Distributed rate limiting and overload protection
- Real-time notifications over WebSockets
- Validation, tests, logs, and production configuration
- Multiple instances behind a load balancer

## Prerequisites

- Current Node.js LTS release
- npm
- Git
- A code editor
- Basic JavaScript knowledge is helpful. The lessons explain Node.js concepts, but they do not replace a complete JavaScript language course.

Check the tools:

~~~powershell
node --version
npm --version
git --version
~~~

See [CURRICULUM.md](CURRICULUM.md) for the full learning path.

## Ground rules for using AI

- Predict first, run second, ask third.
- Never accept code you cannot explain.
- Ask for the smallest useful hint.
- Ask the agent to cite official documentation for version-sensitive claims.
- After a solution works, explain it back to the agent in your own words.
- Keep mistakes. They are excellent study material, annoyingly enough.
- For every feature, ask what happens twice, concurrently, after restart, or during dependency failure.
