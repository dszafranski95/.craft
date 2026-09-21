# 00-START.md — Craft Code Standard: entry point

**Version 2.2** · Read this file first. It routes everything else.

---

## For the human: how to use this

1. Drop the `.craft/` folder into the root of your repository.
2. At the start of every new chat or context window, paste the prompt in §1 below.
3. Work normally.

That is the whole setup. There is nothing to configure and nothing to install.

---

# 1. The bootstrap prompt

Copy this verbatim. Paste it as the first message of any new session, before your actual task.

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

**Shorter variant** for tools that already keep `.craft/` in context automatically:

```text
Follow the Craft Code Standard in `.craft/`. Route per `.craft/00-START.md` §3.
Rules override your defaults. Acknowledge in one line, then wait.
```

---

# 2. Bootstrap contract — what the model does on receiving that prompt

On receipt, and **before** answering any task:

1. Read `00-START.md` and `01-CARD.md`. Nothing else yet.
2. Reply with exactly one line: `CRAFT: ready · 00-START, 01-CARD`.
3. Do not summarise the standard, do not list its rules, do not explain what you are about to do.
4. On the first real task, run §3 and declare the routing decision in one line.
5. Treat the routing decision as **per-task**, not per-session. A new task re-runs §3.

If `.craft/` cannot be read, say so in one line and stop. Never reconstruct the standard from memory and present that as compliance.

---

# 3. Routing procedure

The standard is six files plus one optional folder. **Do not load all of them by default** — loading ~2,350 lines for a typo is exactly the ceremony this standard rejects.

| File | Size | Role | Loaded |
|---|---|---|---|
| `00-START.md` | ~200 | routing only | always |
| `01-CARD.md` | ~110 | laws, STOP list, authority order, decision loop, evidence levels, section index | always |
| `02-PROTOCOL.md` | ~470 | how to work: recon, implementation, AS-01…AS-20, reporting | when changing code |
| `03-GATE.md` | ~530 | how to prove it: triage, gates, evidence table, done | before claiming done |
| `04-STANDARD.md` | ~670 | what good code is: complexity, canon, principles | design & review |
| `05-DECIDE.md` | ~370 | how to spend thinking: fast vs deep, anti-loop, autonomy | when looping or escalating |
| `decide/` | ~830 | runtime contract for a fast decision provider | only when building one |

Deterministic. Follow in order; stop at the first match.

### Step 1 — Does this task change code in this repository?

**No** → load nothing further. Answer normally.
Explaining a concept, a shell command, a library question, a one-line factual answer, general discussion — the Craft files do not apply and must not be loaded.

**Yes** → continue.

### Step 2 — Does any mandatory trigger fire?

If **any** of these is true, the change is **class R** regardless of how small it looks:

- touches auth, authorisation, payments, personal/sensitive data, cryptography, secrets
- database migration, schema change, deletion, or any irreversible operation
- changes a public contract: API, wire format, schema, CLI, file format, published type
- introduces a new dependency
- introduces a new abstraction, layer, module, package or service
- concurrency, async ordering, retries, or distributed state
- a hot path with a stated performance budget
- a STOP condition from `01-CARD` fired
- you are about to suppress a compiler, type, lint or security check
- the human asked for a review, an audit, or "is this good code?"

### Step 3 — Otherwise classify the change

| Class | What it is | Examples |
|---|---|---|
| **T** | no behaviour change, no blast radius | comment, copy string, formatting inside touched lines, obvious dead-code removal |
| **S** | ordinary behaviour change, reversible, local | bug fix, new endpoint, business logic, compatible schema addition, refactoring |
| **R** | anything from Step 2 | — |

**If unsure between two classes, choose the higher one.** Class is decided by blast radius and reversibility, never by diff size: one character in an authorisation predicate is class R.

### Step 4 — Load per class

| Class | Load before writing | Load before completing |
|---|---|---|
| **T** | card only | nothing — self-check against the card |
| **S** | card + `02-PROTOCOL.md` | `03-GATE.md` §1.1 → applicable gates → §19 |
| **R** | card + `02-PROTOCOL.md` + `04-STANDARD.md` | `03-GATE.md` in full, plus §20 review prompt |

Design conversations, architecture questions and "should we do X?" load `04-STANDARD.md` regardless of class — that is the file that answers *why*.

`05-DECIDE.md` is **not** part of any class tier. The decision loop it describes is on the card; load the file itself only when the loop is failing — you are repeating an action, a check keeps failing the same way, five steps have passed with no new fact — or when the human asks about agent autonomy. Load `decide/` only when building an agent runtime.

### Step 5 — Declare the decision in one line

```text
CRAFT: class S · loaded 02-PROTOCOL · 03-GATE before completion
```

That line is the whole ceremony. No preamble, no explanation.

### Step 6 — Re-route mid-task if a trigger fires

Classification is provisional. The moment recon reveals a trigger from Step 2 — the fix turns out to need a schema change, or you are about to add a dependency — **escalate immediately, load the missing file, and say so in one line**:

```text
CRAFT: escalating S → R (schema change) · loading 04-STANDARD
```

Never continue at a lower tier because you already started at one.

---

# 4. Context economy

**Do not re-read what is already in this context window.** If `02-PROTOCOL.md` was loaded earlier in this session, use it; do not load it again. If you are unsure whether you read a file in *this* context — you did not. Check rather than assume.

**Prefer section reads over file reads.** The card's index maps questions to anchors (`STANDARD` §5.6, `PROTOCOL` AS-12, `GATE` §8). Reading one section is the normal move; reading a whole file is for class R and for review.

**Batch independent probes.** When the next few read-only actions do not depend on each other's results, issue them in one step. Collapsing *think → read → think → read* into *think → read ×3 → think* is the single largest saving available to an agent, and it costs nothing in rigour.

**Working from memory is allowed for the card's ten laws only.** Everything below that level — a specific gate, an AS-rule, a threshold, the report format — is read, not recalled. If you cite a rule, cite its anchor so it can be checked.

**Never reconstruct a missing file from memory.** Say which file you could not load, in one line, and work from the card.

---

# 5. Mandatory behaviour

Independent of routing:

- Inspect before editing; search for existing patterns before creating new ones.
- Model data and invariants before adding architectural ceremony.
- Prefer the smallest **coherent** solution — not the smallest hack.
- Do not invent requirements, APIs, env vars, schema fields or fallback behaviour.
- Do not weaken types, tests, linting or security checks to obtain a green result.
- Do not create abstractions merely to remove visual duplication.
- Every new boundary must hide more than its interface costs.
- Do not refactor unrelated code, and do not fix unrelated problems silently — report them.
- Verify claims with tools; state the evidence level. If verification cannot be run, say so.
- Do not deliberate over a question a tool already answered, and do not repeat an action that produced no new information — replan instead (`05-DECIDE.md` §8).
- Review the final diff line by line.

## Completion

Do not declare a task complete until the applicable BLOCKER items in `03-GATE.md` pass, and report in the format on the card.

A passing test suite does not override poor architecture, security issues, false abstractions or hidden failures.

AI-generated code is held to exactly the same engineering standard as human-written code.

---

# 6. Project-local section

<!-- Fill in per repository. Everything above is shared and should not be edited per project. -->

```text
Stack:
Entry point:
Canonical domain types:
Storage / schema:
Test command:
Lint / typecheck command:
Build command:
CI definition:
Local conventions that override generic style:
Areas that are always class R here:
Known debt that is deliberate (do not "fix" opportunistically):
```
