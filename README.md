# Craft Code Standard

Give your AI coding assistant one folder of clear engineering rules.

Craft helps an AI understand a repository before it edits code, make the smallest correct change, and prove that the result works. It works with any coding assistant that can read files in your project.

There is nothing to install, build, or run.

## Start in 30 seconds

1. Download this repository as a ZIP.
2. Open the ZIP.
3. Copy the inner `.craft` folder into the root of your project.
4. Start a new chat with your AI coding assistant.
5. Paste the prompt below.

Your project should look like this:

```text
your-project/
├── .craft/
│   ├── 00-START.md
│   ├── 01-CARD.md
│   ├── 02-PROTOCOL.md
│   ├── 03-GATE.md
│   └── 04-STANDARD.md
├── src/
└── ...
```

## Copy this prompt

Paste this as the first message in every new chat or context window:

```text
This project follows the Craft Code Standard. The rules live in the `.craft/` folder
at the repository root. Read them now, before answering anything.

1. Read `.craft/00-START.md` and `.craft/01-CARD.md` in full.
2. Follow the routing procedure in 00-START §3 to decide which of the remaining
   files to load for the task at hand. Do not load all of them by default.
3. Re-run that routing decision for every new task in this session, not just the first.

These rules are mandatory. They override your own defaults, habits and stylistic
instincts. Where a rule and your instinct disagree, the rule wins. If you are about
to break one, stop and say so instead of doing it quietly.

If `.craft/` is missing or unreadable, tell me in one line. Do not proceed as if
you had read it.

Acknowledge with one line only — `CRAFT: ready · <files read>` — then wait for my
task. Do not summarise the rules back to me.
```

The assistant should reply with one short line:

```text
CRAFT: ready · 00-START, 01-CARD
```

Now ask it to do your task normally.

## What Craft does

Craft tells the assistant to:

- inspect the repository before writing code;
- understand data, contracts, and existing patterns;
- avoid unnecessary abstractions and dependencies;
- stop instead of guessing about dangerous changes;
- test important changes;
- show real evidence before saying the work is done;
- keep unrelated code untouched.

## Why there are five files

The assistant does not need every rule for every task. The numbered files make the reading order obvious and keep the context small.

| File | Purpose | When it is read |
|---|---|---|
| [`.craft/00-START.md`](.craft/00-START.md) | Start here and choose what to read | Always |
| [`.craft/01-CARD.md`](.craft/01-CARD.md) | Short list of the most important rules | Always |
| [`.craft/02-PROTOCOL.md`](.craft/02-PROTOCOL.md) | How to inspect, change, and report work | When changing code |
| [`.craft/03-GATE.md`](.craft/03-GATE.md) | How to verify the result | Before finishing a code change |
| [`.craft/04-STANDARD.md`](.craft/04-STANDARD.md) | Full design and code-quality standard | Risky changes, design, and reviews |

## How task routing works

For every new task, the assistant chooses a level:

- **T — tiny:** text, comments, formatting, or another change with no behavior change;
- **S — standard:** a normal local code change that is easy to reverse;
- **R — risky:** security, payments, personal data, migrations, public APIs, new dependencies, concurrency, or architecture.

Small tasks stay fast. Risky tasks get more checks.

## Optional short prompt

If your tool already keeps the `.craft` folder in context, use this:

```text
Follow the Craft Code Standard in `.craft/`. Route per `.craft/00-START.md` §3.
Rules override your defaults. Acknowledge in one line, then wait.
```

## Optional automatic setup

Some coding tools automatically read a project instruction file. Put the short prompt above in that file.

For Claude Code, create `CLAUDE.md` in your project root with this line:

```text
Follow the Craft Code Standard in .craft/. Route per .craft/00-START.md §3.
```

This is optional. Copying the main prompt into a new chat works everywhere.

## Add your project commands

Open `.craft/00-START.md` and find **Project-local section** at the bottom. Add the real commands for your project, for example:

```text
Stack: TypeScript, Node.js
Entry point: src/index.ts
Test command: npm test
Lint / typecheck command: npm run lint && npm run typecheck
Build command: npm run build
CI definition: .github/workflows/ci.yml
```

Do not invent commands. Use only commands that really work in your repository.

## Common questions

### Do I need to install anything?

No. Craft is only a folder of Markdown files.

### Do I need to run a command?

No. Copy the folder and paste the prompt.

### Do I paste the prompt before every task?

Paste it once at the start of every new chat or context window. The assistant must choose the task level again for every new task in that chat.

### Does Craft replace tests, CI, or code review?

No. It tells the assistant to use them correctly and to be honest when a check was not run.

### Can I change the rules for one project?

Yes. Use the Project-local section in `00-START.md` for repository-specific commands and conventions.

## Updating

To update Craft later, download the newest version and replace the five files in your project's `.craft` folder. Keep a copy of your Project-local section first, then add it back to the new `00-START.md`.

That is all: copy one folder, paste one prompt, and work normally.
