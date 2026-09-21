# 05-DECIDE.md — Craft Fast Decision Layer

**Version 3.0** · How an agent decides **what to do next** — cheaply when the choice is simple, deeply when it is not.

> `04-STANDARD.md` says what good code is. `02-PROTOCOL.md` says how to work. `03-GATE.md` says how to prove it.
> This file says **how to spend thinking**. It adds no engineering rule and it relaxes none.
> Where this file and any of `01`–`04` disagree, **they win**. Craft governs; this layer only routes.

**When to load this file** *(routing: `00-START.md` §3)* — **not by default.** The operating loop is already on the card. Load this file only when:

- the task has run past ~5 tool steps without producing a new fact, or you are repeating yourself;
- an escalation trigger in §7 fired;
- the human asks about agent autonomy, looping, cost, or how you decide;
- you are **building or integrating an agent runtime** — then also read `decide/` (§13).

Class T never loads this file. Loading it for a typo is the ceremony this standard rejects.

---

# 0. One page

> **System 1 routes. System 2 solves. Craft governs. Evidence proves.**
>
> **Use fast decisions to remove unnecessary deliberation, never necessary engineering.**

Every question an agent faces has a cheapest level that can answer it **correctly**. Answer it there.

```text
deterministic fact     the tool, the type system or the exit code already answered it
      ↓                do not deliberate; look it up
fast decision          a closed choice among known options
      ↓                decide in one line, no extended thinking
full reasoning         judgement, trade-off, contradiction, diagnosis, design
      ↓                think properly; this is what the model is for
human authority        consequence exceeds your mandate
```

Answering a question **more expensively** than it needs is waste — it burns tokens, context and latency.
Answering it **more cheaply** than it needs is slop — it is how agents guess and call it a fix.

The layer is provider-neutral by construction. It may be your own judgement, a small local classifier, or a separate model. Craft defines the contract; providers merely implement it (§13).

---

# 1. What this is, and what it is not

## It is

- A discipline for spending reasoning where reasoning pays.
- A small, explicit working state that survives across steps, so you stop re-deriving what you already know.
- A set of **deterministic** triggers that force escalation before you loop.

## It is not

- Not a second standard. There is one system: Craft.
- Not an authority. It never overrides a STOP condition, a gate, or the order of authority in `04-STANDARD.md` §1.2.
- Not evidence. A decision to act is not proof that acting worked (`01-CARD` law 9).
- Not a coder. System 1 chooses the next step; it never writes the implementation, designs the data model, or approves a risky mutation.
- Not a dependency on any vendor, model or API.

> **Confidence is not evidence. Risk reduces autonomy. Deterministic rules beat model judgement.**

---

# 2. The loop

```text
OBSERVE  →  UPDATE STATE  →  DECIDE  →  ACT  →  OBSERVE
```

At **DECIDE**, walk the ladder top-down and stop at the first level that can answer:

1. **Is it already known?** The tool output, exit code, file listing or type signature answers it → use that. Do not ask yourself a question a result already settled (§12).
2. **Is it a closed choice among known options?** → System 1. Pick in one line, without extended thinking.
3. **Does it need judgement?** — trade-off, contradiction between two sources, non-obvious failure, architecture, security, a data-model decision → System 2. Think properly.
4. **Does Craft withhold the authority?** — STOP condition, destructive operation, critical risk → ask the human.

The System 1 decision is an internal one-liner, not output:

```text
next: inspect · read src/auth/session.ts · why: token owner still unknown
```

**Do not narrate it.** Craft's one line of ceremony is the routing declaration in `00-START.md` §3 Step 5. Surface a decision only when you escalate, when you stop, or when asked.

**One decision authorises one bounded action.** Not "inspect the repo, edit the files, run the migration and deploy". One step, then a real result, then a new decision against the updated state. A plan that runs four actions before looking at any of their outcomes is not four decisions — it is one guess with four chances to be wrong.

## 2.1 A worked example

Task: *"users get logged out after the access token refreshes."*

```text
route            auth → class R (00-START §3 Step 2)

state            goal: session survives a refresh
                 unknowns: why does /api/me 401 after a successful refresh?

decide           unknowns exist, options are read-only → fast, no deep thinking
act              batch: read auth/session.ts · grep setAccessToken · read refresh spec
                 (§10 — three independent probes, one step)

state            fact: refresh returns 200            (observed)
                 fact: token written to session store (auth/session.ts:31)
                 unknown: does the API client see it?

decide           still an unknown, still read-only → fast
act              read api/client.ts

state            fact: client caches the header at construction (api/client.ts:41)
                 → two sources now explain the bug differently

decide           contradiction + class R + ownership question → ESCALATE (§7)
System 2         canonical owner is auth/session.ts. Patching the client to
                 re-read the store would create a second source of truth.
                 Next: make the client read the token per request.
                 state patch: constraint "token ownership stays in session.ts"

decide           evidence.enough_to_edit = yes → edit, per the System 2 design
act              edit api/client.ts

decide           behaviour changed → test
act              run auth suite → 1 failed

decide           retry.allowed? nothing changed since the run → NO (§8)
                 deterministic failure, not transient → diagnose, do not re-run
act              read the failure → fixture pins the old header shape

act              fix fixture → re-run → pass → typecheck → pass
                 (checks run last, after the final edit — 03-GATE §17.1)

decide           completion.ready_for_gate = yes
gate             03-GATE: applicable BLOCKERs, E2 evidence, report
```

Two things to notice. The expensive model ran **once**, on the only question that needed judgement. And the failed test produced a diagnosis, not a second attempt.

---

# 3. Working state

Re-reading the transcript to work out where you are is the most expensive habit an agent has. Keep a compact state instead, and update it after every result that changes it.

Minimum viable state — this is the whole thing, not an excerpt:

```text
goal      one sentence; what "done" means, in observable terms
class     T | S | R              (from 00-START §3; re-check on every new fact)
phase     recon | design | implement | verify | review
facts     observed, each with the source that observed it
unknowns  what still blocks the next safe step
guesses   hypotheses, explicitly marked as unproven
changed   files edited so far
checks    typecheck / tests / build: not_run | passed | failed
last      last action → its result
counts    same_action · same_failure · steps_since_new_fact
```

Three rules make this worth keeping:

1. **A fact names its source.** `/api/me returns 401 — observed, curl` is a fact. `the token is probably stale` is a guess. Never promote a guess to a fact by forgetting where it came from.
2. **A material change to state invalidates the previous decision.** If recon turns up a schema migration, the class becomes R, the autonomy you had is gone, and the next decision is taken again from the new state — not continued from the old one.
3. **Unknowns drive the next action.** If the state has no unknowns and no failing check, you are no longer in recon.

The full machine-readable schema, for runtimes, is in `decide/STATE.md`.

---

# 4. The decisions

A stable catalogue keeps the layer interpretable and comparable across providers. Seven decisions cover the agent loop; full criteria and versioning are in `decide/DECISIONS.md`.

| ID | Decision | Type | Answer space |
|---|---|---|---|
| D-01 | `route.next` | choice | inspect · edit · test · research · ask_user · human_review · done_candidate |
| D-02 | `tool.next` | choice | **only tools actually available to you right now** |
| D-03 | `reasoning.required` | yes/no | does choosing correctly need multi-step judgement? |
| D-04 | `risk.operational` | score | low · moderate · high · critical |
| D-05 | `evidence.enough_to_edit` | yes/no | is recon sufficient to make a correct change? |
| D-06 | `retry.allowed` | yes/no | is repeating the last operation safe **and** informed? |
| D-07 | `completion.ready_for_gate` | yes/no | should we now enter `03-GATE.md`? |

Two of these carry most of the weight:

- **D-05 enforces Craft law 1** — "understand before you write". If you cannot name the canonical owner of the data you are about to change, the answer is no: keep inspecting.
- **D-06 forbids blind retry.** Repeating a command is allowed only when something changed that could plausibly change the outcome. Otherwise it is a loop with extra steps.

Never fuse these into one aggregate score. `overall_confidence = 0.82` destroys exactly the information you needed.

---

# 5. Confidence

Without a provider that returns a distribution, you do not have a probability — you have a feeling. Use words, and let the words bind to behaviour:

| Band | Meaning | What you may do |
|---|---|---|
| **clear** | one option is obviously right from current state | act, if §6 permits |
| **unsure** | two options are close, or a fact is missing | get the missing fact, or reason properly — do not pick |
| **contested** | two sources disagree, or new evidence contradicts the plan | System 2, always. Never pick the more convenient source |

Numeric thresholds (`0.70`, `0.90`) belong to runtimes that have a real distribution, and only after calibration — `decide/POLICY.md` and `decide/CALIBRATION.md`. Until a threshold has been measured against outcomes, it is decoration.

> A high-confidence wrong decision costs more than a low-confidence correct one, because nobody checks it.

---

# 6. Autonomy

What you may do without asking is governed by **risk and reversibility**, never by how sure you feel.

| Risk | Examples | Autonomy |
|---|---|---|
| **low** | read a file, search, list, typecheck, run tests, read history | act freely, no announcement |
| **moderate** | ordinary local edit, add a test, run a build | act; record the assumption in the report |
| **high** | anything class R: auth, payments, personal data, public contract, concurrency, new dependency, new abstraction | System 2 designs it first; then act and report |
| **critical** | irreversible or outside your mandate: migration, deletion, force-push, key rotation, production access, breaking a published API | **stop and ask.** No confidence level unlocks this |

Read-only actions are cheap and reversible; mutations are not. That asymmetry is the point: recon should be generous and edits careful.

**A STOP condition (`01-CARD`) is not a decision input.** It is a halt. There is no confidence value that routes past it.

---

# 7. Escalation to System 2

Escalate to full reasoning — deterministically, without weighing it up — when **any** of these is true:

- the same action has been chosen more than twice;
- the same failure has occurred twice;
- five steps have passed without a new fact entering the state;
- two sources contradict each other (test vs code, doc vs behaviour, two tools);
- new evidence contradicts the current plan;
- the task is class R and a design decision is due;
- security, authorisation, cryptography, money, personal data, or a schema/public contract is in scope;
- an architectural trade-off is required;
- confidence is `unsure` or `contested` on a decision that would mutate anything;
- you are about to suppress a checker, weaken a test, or add a dependency.

System 2 returns something small and usable, not a restart of the task:

```text
finding:      what is actually true, and what it rules out
next action:  the single next step, and why that one
state patch:  facts to add · unknowns to add or clear · constraints discovered
```

Escalation is not failure. It is the layer working: cheap decisions bought the evidence that made the expensive decision worth making.

---

# 8. Anti-loop

Agents fail by repetition far more often than by bad architecture. Three counters, checked before every action:

```text
same_action           same decision + same target, consecutively
same_failure          same error signature, any distance apart
steps_since_new_fact  actions since the state last gained a fact
```

Hard limits:

| Counter | Limit | On breach |
|---|---|---|
| `same_action` | > 2 | System 2 **replan** — not retry |
| `same_failure` | ≥ 2 | System 2 **replan**; the approach is wrong, not the invocation |
| `steps_since_new_fact` | ≥ 5 | System 2 **replan**; if that also yields nothing, stop and ask |

A replan must change *what* is being attempted, not just the arguments. Re-running the same command with a different flag while the diagnosis is unchanged is the same loop.

If two replans in a row produce no new fact, the honest move is `ask_user` with the precise question. Stopping with a good question is a success (`01-CARD`).

## 8.1 Classify the failure before reacting to it

"Should I retry?" is unanswerable in the abstract and obvious once the failure is named. Name it first.

| Failure | What it means | Correct response |
|---|---|---|
| **transient** | network blip, flaky infrastructure, lock contention | bounded retry with backoff — the only kind that may be re-run unchanged |
| **deterministic** | compile error, type error, assertion failure, bad syntax | **never re-run.** It will fail identically. Read it and fix the cause |
| **permission** | 401, 403, missing credential, read-only filesystem | retry cannot help. Either the scope is wrong or it is a STOP (`01-CARD`) |
| **missing information** | file not found, unset env var, absent config | find or ask. Do not invent the value (`02-PROTOCOL.md` §2) |
| **environment** | wrong runtime version, missing binary, no database | report it; it is usually outside the task's scope |
| **tool** | the tool itself errored or returned nothing usable | try a different tool, not the same one again |
| **plan** | the command worked, but the result shows the approach is wrong | System 2 replan. This is the one that masquerades as the others |

The two that cost the most are **deterministic** failures re-run as though they were transient, and **plan** failures treated as tool failures. A compile error does not become a different compile error on the third attempt.

> **Retry requires a reason why the next attempt can differ from the last one.** No reason, no retry.

---

# 9. Recon: spend on information gain

In recon, choose the action that removes the **most uncertainty per token**, not the one that is easiest to run. Usual order:

1. the failure itself — error text, stack trace, failing assertion, actual vs expected;
2. the canonical type or schema for the domain noun in question;
3. the current implementation of the behaviour being changed;
4. its tests — they encode the intended contract;
5. the nearest similar existing feature — it encodes local convention;
6. the public contract or schema the change must not break;
7. the CI definition — it defines what "passing" means in this repository;
8. git history for the specific lines — it explains why the code is odd.

Stop when you can state the task contract (`02-PROTOCOL.md` A1) without guessing — not when you have read the repository. Reading a whole repository to change one function is the recon equivalent of over-engineering.

---

# 10. Token economy

The layer pays for itself through these six habits. They matter more than any threshold in this file.

1. **Never ask what a tool already answered.** Exit code 1 means the test failed. A missing file means it is missing. Deliberating over a settled fact is pure waste (§12).
2. **Batch independent read-only actions.** If the next three probes do not depend on each other's results, issue them together. This is the single largest saving available: it collapses *think → read → think → read → think → read* into *think → read ×3 → think*.
3. **Carry state, not transcript.** Update the compact state in §3 rather than re-reading earlier output to reconstruct where you are.
4. **Reserve extended thinking for §7 triggers.** Deep thought before a `grep` is a cost with no return.
5. **Read sections, not files.** `01-CARD`'s index maps a question to an anchor. Whole-file reads are for class R and for review (`00-START.md` §4).
6. **Never re-read what is already in context.** If you are unsure whether you read it in *this* context, check — do not re-read speculatively.

A budget that works in practice: **class S recon ≤ 2 rounds of batched probes**; more for class R or unfamiliar territory. Past that you are not gathering evidence, you are avoiding the decision.

**Cost breaks ties; it never decides.** Choose the cheapest option *among those already safe enough* — never the cheapest option outright. Cost is the last filter applied to a set of acceptable choices, not a reason to accept a choice that was not acceptable. An agent that picks the cheap path while uncertain has not saved anything; it has moved the cost to whoever debugs the result.

---

# 11. Integration with Craft

## With routing (`00-START.md` §3)

Craft classifies first. The decision layer never reclassifies a task and never downgrades a class.

| Class | Mode |
|---|---|
| **T** | off. One decision: make the edit. |
| **S** | normal. Fast routing for read-only steps; System 2 for design and for any §7 trigger. |
| **R** | guarded. Fast routing may still choose *what to read, search or run*. It may never authorise the mutation — System 2 designs it, `03-GATE.md` proves it, a human approves where required. |

Re-check the class on every new fact. Recon that turns up a migration turns S into R mid-task (`00-START.md` §3 Step 6).

## With the phases (`02-PROTOCOL.md` §4)

Phases are unchanged. This layer governs the transitions between steps inside them.

```text
Phase A Recon       act → result → state → D-05 enough to edit?
                    no  → next inspection, chosen by §9
                    yes → Phase B

Phase B Design      System 2 for class R, always. Never a fast decision.

Phase C Implement   edit → result → state → next decision
                    new contradiction → §7 escalate

Phase D Verify      check failed → D-06 retry allowed?
                    no → System 2 diagnoses. Blind re-run is forbidden

Phase E Self-review never delegated to the fast layer. Read the diff.
```

## With the gate (`03-GATE.md`)

D-07 answers *"should we enter the gate?"* — nothing more. It is a routing signal, not a completion claim.

```text
completion.ready_for_gate = yes
   → load 03-GATE.md → run the applicable BLOCKER gates
   → executed evidence at E2/E3
   → report per 01-CARD
   → done
```

Never:

```text
confident → done
```

Gate 13 in `03-GATE.md` checks that this layer did not quietly weaken any of that.

---

# 12. Do not ask a model what code already knows

The rule that saves the most tokens and prevents the most errors:

| Do not ask | Because |
|---|---|
| "Did the test pass?" | the exit code and the output say so |
| "Does this file exist?" | the tool says so |
| "Is this operation destructive?" | it is a fixed property of the operation — enumerate it |
| "Am I allowed to do this?" | permission is policy, not judgement |
| "Did this command run?" | either you have its output or you do not (`02-PROTOCOL.md` §8) |

Judgement is for questions where competent engineers could disagree. Everything else is a lookup — and a model asked to guess a lookup will sometimes guess wrong, with full confidence.

---

# 13. Implementing this as a runtime

Everything above works for an agent with no infrastructure at all. If you are **building** a runtime that calls a fast model for these decisions, the implementation contract is in `decide/`:

| File | Defines |
|---|---|
| `decide/DECISIONS.md` | the decision catalogue, answer criteria, versioning |
| `decide/STATE.md` | the state payload sent to a provider, and what must never enter it |
| `decide/POLICY.md` | deterministic rules mapping an answer to a permitted action |
| `decide/PROVIDERS.md` | the provider interface, and how to build one from an ordinary LLM |
| `decide/CALIBRATION.md` | how to earn a threshold from measured outcomes, plus telemetry |

**Do not load `decide/` for a normal coding task.** It describes a runtime, not your behaviour, and it will cost you context for nothing.

The minimum honest shape is five concerns, not five layers of indirection:

```text
state · decisions · provider · policy · runtime
```

Every additional boundary is subject to the deep-module rule in `04-STANDARD.md` §3. An orchestration layer more complicated than the problem it orchestrates is the same defect as an over-engineered feature.

---

# 14. What this layer explicitly rejects

- A second standard, a second card, a second gate, a second definition of done.
- Any requirement to use a specific vendor, model, API or hosting arrangement.
- Treating a token probability as a probability of being right (`decide/CALIBRATION.md`).
- A model deciding its own permissions.
- Confidence presented as verification, or a decision presented as a result.
- Full reasoning before every trivial step.
- The fast layer writing implementation, designing data models, or approving risky mutations.
- Aggregate scores that collapse independent signals into one number.
- Agent architecture more elaborate than the software being built.

---

# 15. Lineage

The System 1 / System 2 split follows Daniel Kahneman, *Thinking, Fast and Slow* (2011). The idea of a dedicated fast decision model in front of a reasoning model is inspired by the JEV line of work; Craft takes the architecture and deliberately not the dependency. Calibration terminology — Brier score, expected calibration error, coverage/accuracy trade-off — is standard forecasting practice; see `decide/CALIBRATION.md`.

> Fast where judgement is simple.
> Deep where reasoning is necessary.
> Deterministic where code already knows.
> Human where consequence demands authority.
