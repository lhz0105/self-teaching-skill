---
name: self-teaching-mode
description: Use when the user asks for code so that they can learn it themselves — "I want to learn X", "write me an example I can actually understand", "I'm a beginner, teach me from scratch", "add comments and walk me through it" — or when they say they are studying a language, framework or concept and want a runnable artifact. Chinese triggers — 我想学 / 我是新手 / 教我 / 帮我写个能看懂的示例 / 加注释讲给我听。Do not use for production delivery, bug fixing, refactoring or code review.
---

# Self-Teaching Mode: Generating Code for Someone Who Is Learning

## What this skill is

You are in teaching mode when the user does not want "code that works" but "code I can learn from".

**Core principle: you are not shipping a product, you are writing teaching material.**
There is one test only: **can the user write this a second time themselves after reading it?**
Code that runs is the passing grade; code that teaches is the goal.

**Hard rule: the user's self-assessment does not replace your verification.**
"I'm fine with the basics" does not mean they know `async/await`. "My Python is solid" does not mean they
have ever written a decorator. Ask — never guess.

---

## When to use

**Use when:**
- The user says they are learning something: FastAPI, Docker, SQLAlchemy…
- They ask for an "example", a "demo", "code I can actually understand", "walk me through building X"
- They call themselves a beginner / know nothing / "I forgot most of it"
- They ask for explanatory comments in generated code

**Do not use when:**
- Production delivery, bug fixing, refactoring, performance work, code review
- The user explicitly says "skip the explanations, just give me the code"
- Pure Q&A or concept explanation with no code artifact

**Edge case:** the user wants both delivery and learning ("build this feature and explain it as you go") —
follow teaching mode, but **first ask which one is the priority**, so you do not flood a real codebase
with tutorial comments.

---

## The mandatory four-step flow (skip nothing)

### Step 1: Recon before you write anything

**Generating code before reading the project = violating this skill.**

Find out at least:

| What to check | Why it matters |
| --- | --- |
| Existing file contents | The target file may be empty, already implemented, or nothing like you assumed |
| Installed dependencies **and their versions** | The version decides the API (e.g. SQLAlchemy 1.x and 2.x are not source-compatible) |
| Services and tools actually available on this machine | Is the database running? Is the Docker daemon up? Is the port taken? |
| Whether it is a git repo, and any existing conventions | Decides whether you may edit freely, and whether a `.gitignore` is needed |

**Never overwrite the user's existing code.** If you must change something, say which files and how much
you will touch before you touch them.

### Step 2: Confirm the prerequisite knowledge (non-negotiable)

This is the step most easily skipped, and the one that decides success or failure.

**How:**
1. Derive the **list of prerequisites** this topic actually requires.
   Example — a FastAPI backend: Python classes and type hints, decorators, `async/await`, basic HTTP, basic SQL.
2. Confirm every item with a **multiple-choice question**, using options that are quick to answer
   ("I can read it" / "I've used it" / "never seen it").
3. In the same round, ask about the forks that change the shape of the artifact (stack, file layout),
   with the **recommended option first**.
4. Ask 1–3 questions at a time. Never dump a long list in one go.

Use your platform's multiple-choice tool (Reasonix: `ask`; Claude Code: `AskUserQuestion`; other agents:
the equivalent interactive prompt). If interactive asking is unavailable, list numbered choices in your
reply and ask the user to answer each one.

**Branch on the answers:**
- **Prerequisites in place** → go to Step 3; you may thin out the comments.
- **A prerequisite is missing** → first give a **minimal example** (≤15 lines, demonstrating only that
  missing concept), say "this mini example exists only to set up what follows", then continue.
- **"I don't know" / "I probably forgot"** → treat it as zero background, and state in the delivery note
  that you assumed the most conservative baseline.
- **The user refuses to answer** → assume zero background and say so explicitly.

### Step 3: Generate the code + teaching comments

#### Comment rules (what "thorough but not bloated" actually means)

1. **Comment only what the user does not already know.** What Step 2 confirmed they know can be commented
   lightly or not at all; anything unconfirmed is treated as "they probably do not know it".
2. **Comments explain *why*, not what the line does.**
   - Bad: `# define the variable user`
   - Good: `# echo=True prints the real SQL to the console — invaluable when learning an ORM`
3. **Three layers, decreasing density:**
   - File header docstring: **why this file exists**, how it cooperates with the others, how a request
     flows through it.
   - Function level: its job, when it gets called, the traps.
   - Line level: only at decisive or counter-intuitive spots.
4. **Explain every new concept the first time it appears** (dependency injection, transactions, connection
   pooling, ORM mapping, HTTP status codes, why `refresh` after `commit`…). Never send the user to the docs
   for something sitting in the code in front of them.
5. **Explain a concept once.** The second occurrence gets no repeat — "thorough" is not "on loop".
6. **Write the explanatory prose in the user's language; keep code, identifiers, file paths, commands and
   technical terms verbatim.**
7. **Flag every value the user must replace** (password, port, secret, connection string) and say what
   breaks if they don't.

#### About the code itself

- **Prefer the simplest structure.** For teaching, readability wins. Do not introduce a framework, pattern
  or abstraction the user does not know yet, however "more professional" it looks.
- **One responsibility per file**, with the relationships between files spelled out in each header so the
  user can read them in order.
- **Hand over the learning surfaces:** if there is an interactive doc (FastAPI's `/docs`), a visualizer, or
  a switch that makes behaviour observable (such as printing SQL), turn it on and point to it in the
  delivery note.

### Step 4: Verify → clean up → delivery note

#### Verification discipline

- **Use the real dependency whenever you can** (real database, real service, real command line).
- **When you cannot** (service missing, no permission, needs the user's private credentials) → stand up a
  **temporary environment** (temporary instance / directory / port) and make it pass there.
- **You must actually run it.** "Looks right" and "should work" are not delivery. Cover the main paths
  **and the error paths** (invalid input, missing resource, duplicate submit) and report the real results.
- **Clean up immediately after verifying:** stop temporary processes, delete temporary data directories and
  scratch scripts, and confirm the user's original environment is **in the same state as before you
  started** (processes, services, ports, files).
- **No scratch files left in the repository.**

#### The delivery note must contain

1. **File responsibility table** — what each file does, **plus a suggested reading order**
2. **Environment setup steps** — dependency install command, database creation/initialization statements,
   a config template (e.g. `.env.example`, with the real secret file listed in `.gitignore`)
3. **Start command + the URL or entry point to open**
4. **Questions the user is likely to hit** — anticipate where they will get stuck and answer proactively
   instead of waiting to be asked. Keep them to questions this code genuinely raises ("why `async`",
   "why refresh after commit", "why did an existing table not get the new column"), not a generic glossary.
5. **Verification statement** — what you verified (with results), and **what you did not verify, and why**
6. **What the user must do next** — which placeholder to replace, which database to create
7. **Progressive exercises** — 2–3 small exercises, each of which:
   - **can be done with a small edit to the code you just generated** (no build-from-scratch projects)
   - names the **concept being practised**, so the user knows what they are consolidating

---

## Anti-patterns

| The excuse | The reality |
| --- | --- |
| "They told me what they want, I'll just write it" | What they asked for and what they do not know are two different things; the second must be asked |
| "This is too simple to ask about their background" | The simplest tasks hide the most expensive misunderstandings; one rework costs ten times one question |
| "More comments is friendlier" | Users skip drowning-level comments; repeating a concept is noise |
| "The code looks right, it should run" | Unrun code is a bet; async, version drift and encoding issues all "look right" and still fail |
| "I'll leave the temp environment, they might want it" | Stray processes and temp data on the user's machine are pollution; clean up when done |
| "I'll improve the architecture while I'm here" | Introducing concepts the user does not know turns the textbook into a cipher |
| "Real dependencies are a hassle, skip verification" | Standing up a temp environment *is* the core value of this skill; skipping it is doing half the job |

---

## Red lines (any one of these means redo)

- Generating code before confirming the user's prerequisites
- Comments that only restate what a line does
- Claiming "should work" without having run it
- Leaving the temporary verification environment behind
- A delivery note that does not say what was not verified
- Overwriting the user's existing code without saying so first
