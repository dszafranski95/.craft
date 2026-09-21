# decide/DECISIONS.md — the decision catalogue

**Version 3.0** · Implementation contract for the Fast Decision Layer. Concept and agent behaviour: `../05-DECIDE.md`.

> **Do not load this file for a normal coding task.** It specifies a runtime, not your behaviour.
> Read it when you are building or integrating a fast decision provider.

---

# 1. What a decision is

A decision is a **small, closed question with a fixed answer space**, answered from the working state alone (`STATE.md`) — no repository access, no transcript, no tool use.

Every decision has:

```text
id          stable identifier, versioned           route.next/v1
type        choice | yes_no | score
answers     the complete, fixed answer space
criteria    one unambiguous sentence per answer
input       the state fields the answer may depend on
default     the answer to assume when the provider fails or is unavailable
```

The **default must be the conservative answer** — the one that costs tokens rather than correctness. A provider timeout must never open autonomy.

Three types, and nothing else:

| Type | Returns | Use for |
|---|---|---|
| `choice` | exactly one answer from the set | "which of these next?" |
| `yes_no` | one of two answers | "is this condition met?" |
| `score` | one ordinal band | "how much of X?" — never a continuous number |

A `score` is an ordinal band with named levels, not a number. `risk = 2.7` means nothing and cannot be audited.

---

# 2. Rules that apply to every decision

1. **The answer space is closed and exhaustive.** A provider that cannot answer returns the default; it never invents an answer.
2. **The answer space is real.** `tool.next` is restricted to the tools the runtime can actually invoke *at that moment*. Offering an unavailable tool guarantees an unexecutable answer.
3. **One decision asks one thing.** Do not fuse `route.next` and `risk.operational` into a single call to save a round trip. Independent signals must stay separable (`PROVIDERS.md` §5).
4. **No aggregate score.** Never derive `overall_agent_confidence`. Collapsing the signals destroys the information the policy needs.
5. **Criteria are behavioural, not vibes.** "Use `inspect` when the next safe change cannot be named from current evidence" is a criterion. "Use `inspect` when more context would be helpful" is not.
6. **A decision never mutates state.** It reads state and returns an answer. The runtime applies the consequence (`POLICY.md`).
7. **Deterministic questions are not decisions.** If the runtime can compute the answer, it computes it (`../05-DECIDE.md` §12).
8. **One decision authorises at most one bounded action.** Never a plan of several steps executed before any of their results are seen. The next decision is taken against the state the previous action actually produced (`../05-DECIDE.md` §2), and anything that mutates additionally needs an action envelope (`POLICY.md` §5).

---

# 3. The catalogue

## D-01 · `route.next/v1` — choice

What kind of step should happen next?

| Answer | Criterion |
|---|---|
| `inspect` | further repository evidence is needed before the next safe coherent change can be named |
| `edit` | the change is understood, its location is known, and the evidence to make it correctly is present |
| `test` | behaviour has changed, or a hypothesis about behaviour needs an executed check |
| `research` | the blocker is outside this repository: a library contract, a protocol, an external API |
| `ask_user` | a STOP condition applies, or ambiguity touches data model, public contract, authorisation, money, or personal data |
| `human_review` | the work is done but risk requires human approval before it lands |
| `done_candidate` | the requested behaviour appears satisfied and the completion gate should now run |

`done_candidate` is a **routing signal**, never a completion claim. See D-07.

**Default:** `inspect`. Gathering evidence is the cheapest way to be wrong.

---

## D-02 · `tool.next/v1` — choice

Which tool executes the step chosen by D-01?

Candidate answers, filtered at call time to what the runtime actually exposes:

```text
none · read_file · search_repo · list_dir · git_history
edit_file · terminal · tests · typecheck · lint · build · web
```

| Rule | Reason |
|---|---|
| The set is generated from the live tool registry | the model must not select a tool that does not exist |
| Mutating tools are excluded unless `POLICY.md` has already permitted mutation | prevents the model from selecting its own privileges |
| `none` is always available | "no tool needed" must be expressible |

**Default:** `none` — hand control back to the runtime rather than guess.

---

## D-03 · `reasoning.required/v1` — yes_no

> Does selecting the next correct action require multi-step reasoning, architectural judgement, resolution of contradictory evidence, a material trade-off, or diagnosis of a non-obvious failure?

`yes` routes to System 2 before anything else happens.

This decision is **advisory only in the upward direction**. A `no` never suppresses an escalation that `POLICY.md` already requires — the deterministic triggers win.

**Default:** `yes`. Thinking unnecessarily costs tokens; not thinking when required costs correctness.

---

## D-04 · `risk.operational/v1` — score

How much damage could the next action do if it is wrong?

| Band | Level | Meaning |
|---|---|---|
| 0 | `low` | read-only, or trivially reversible; no persistent effect |
| 1 | `moderate` | ordinary local code change, reversible by revert |
| 2 | `high` | class-R territory: auth, payments, personal data, public contract, concurrency, new dependency, new abstraction |
| 3 | `critical` | irreversible or outside mandate: migration, deletion, force-push, key rotation, production access, breaking a published contract |

Risk is a property of the **action and its blast radius**, not of the model's certainty.

Where the runtime can determine the band deterministically — the tool is known to be destructive, the path matches a protected glob — **it must do so and not ask.** The model answers only the residual cases.

**Default:** `high`.

---

## D-05 · `evidence.enough_to_edit/v1` — yes_no

> Has enough repository evidence been gathered to make a correct change, rather than a plausible one?

`yes` requires all of:

- the canonical owner of the data being changed is identified;
- the existing implementation of the behaviour has been read;
- the relevant tests have been read;
- the public contract the change must not break is known, or known not to exist;
- no unknown in the state blocks the change.

This decision is the mechanical form of Craft law 1 — *understand before you write*. It is the single highest-value question in the catalogue: a premature `yes` here produces the plausible-but-wrong change that Craft exists to prevent.

**Default:** `no`.

---

## D-06 · `retry.allowed/v1` — yes_no

> Is repeating the last operation both safe and informed?

`yes` requires **both**:

1. the operation is idempotent or its side effects are known and acceptable; **and**
2. something has changed that could plausibly change the outcome — a file edited, an environment variable set, a dependency installed.

If nothing changed, the answer is `no` regardless of how transient the failure looked. A retry with an unchanged diagnosis is a loop.

**Classify the failure first.** The answer follows almost mechanically from the failure class — transient failures may be re-run, deterministic ones never can. The taxonomy is in `../05-DECIDE.md` §8.1, and wherever the runtime can classify a failure from its exit code or error signature, **it must do so and not ask** (§2 rule 7).

**Default:** `no`.

---

## D-07 · `completion.ready_for_gate/v1` — yes_no

> Should the Craft completion gate now run?

It does **not** mean the task is complete. It means the work looks finished enough that verification is worth its cost.

```text
yes → 03-GATE.md runs the applicable BLOCKER gates
    → executed evidence at E2/E3
    → only then: done
```

A runtime that treats `yes` as completion has broken the standard. `../03-GATE.md` decides completion; this decision only decides when to ask it.

**Default:** `no`.

---

# 4. Versioning

Decision semantics must be stable, because calibration data is only valid for the exact question that produced it.

```text
route.next/v1        criteria as published above
route.next/v2        criteria materially changed → recalibrate
```

| Change | Requires |
|---|---|
| wording clarified, meaning identical | same version, note in changelog |
| an answer added or removed | **new version** |
| a criterion's boundary moved | **new version** |
| the input state fields changed | **new version** |

On a new version: existing thresholds are void until re-measured (`CALIBRATION.md` §5). Carrying a `v1` threshold into `v2` is the most common way calibrated systems quietly stop being calibrated.

Every telemetry record stores the decision id **with its version**. Without it the history is unusable.

---

# 5. Adding a decision

Do not. Not until the existing seven are calibrated and a concrete failure shows the gap.

A new decision is justified only when all four hold:

1. the question recurs often enough to matter;
2. the runtime cannot answer it deterministically;
3. its answer changes what the agent does next, not merely what it reports;
4. it cannot be expressed as an answer within an existing decision.

Otherwise it is speculative generality in a new costume (`../04-STANDARD.md` §5.7).
