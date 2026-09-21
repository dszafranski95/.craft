# decide/STATE.md — the decision state

**Version 3.0** · Implementation contract for the Fast Decision Layer. Concept and agent behaviour: `../05-DECIDE.md`.

> **Do not load this file for a normal coding task.** It specifies a runtime, not your behaviour.

---

# 1. Why state exists

A fast decision must be answerable from a payload measured in **hundreds of tokens**, not from a transcript measured in tens of thousands. If the classifier needs the conversation, it is not a fast layer — it is the expensive model with extra steps.

State is the agent's working memory, maintained deliberately:

```text
transcript   everything that was said            grows without bound, mostly stale
context      what the model currently holds      expensive, evicted unpredictably
state        what is currently true              small, current, explicitly maintained
```

Target: **200–1,500 tokens.** If state exceeds that, the agent is hoarding, not tracking — see §6.

---

# 2. Schema

```json
{
  "task": {
    "requested": "Fix login being lost after access-token refresh",
    "success": "User remains authenticated after a refresh cycle",
    "out_of_scope": ["session UI", "logout flow"]
  },

  "craft": {
    "class": "R",
    "class_reason": "touches authentication",
    "stop": false
  },

  "phase": "recon",

  "facts": [
    { "claim": "refresh endpoint returns 200", "source": "curl, observed" },
    { "claim": "/api/me returns 401 afterwards", "source": "curl, observed" }
  ],

  "unknowns": [
    "whether the refreshed token reaches the API client"
  ],

  "hypotheses": [
    { "claim": "api client caches the Authorization header", "status": "unproven" }
  ],

  "constraints": [
    "session shape is a published contract; cannot change"
  ],

  "evidence": [
    "auth/session.ts read",
    "refresh-token.spec.ts read"
  ],

  "changed_files": [],

  "verification": {
    "typecheck": "not_run",
    "tests": "not_run",
    "build": "not_run"
  },

  "last_action": "search_repo setAccessToken",
  "last_result": "defined once in auth/session.ts",

  "counters": {
    "same_action": 0,
    "same_failure": 0,
    "steps_since_new_fact": 2
  },

  "tools_available": ["read_file", "search_repo", "git_history", "tests", "typecheck"]
}
```

---

# 3. The distinction that matters

State must keep these four apart. Collapsing them is how an agent starts believing its own guesses.

| Field | Means | Admission rule |
|---|---|---|
| `facts` | observed | requires a `source`: a command run, a file read, an output seen |
| `hypotheses` | believed, not shown | must carry `status: unproven` until observed |
| `unknowns` | known gaps | removed only when a fact answers them |
| `constraints` | must not be violated | from public contracts, requirements, project convention |

**A hypothesis is promoted to a fact only by an observation, never by the passage of steps.** This is Craft law 9 — *no claim without evidence* — expressed as a data-structure invariant, which is how it survives a long session.

`verification` is likewise **written only by the runtime**, from real exit codes. A model must never be able to set `"tests": "passed"`. That is the difference between a verification record and a wish.

## Working state is not the evidence log

Two different things, kept apart:

| | `evidence` — the observation log | `facts` / `hypotheses` — working state |
|---|---|---|
| Contains | what was actually observed, and when | what is currently believed to be true |
| Mutability | append-only; entries are never edited | revised as understanding changes |
| On a contradiction | keeps both observations | the belief is corrected or dropped |
| On a code change | stays — it is history | entries covering the changed code expire (§4.1) |

The evidence log is the audit trail: it answers *"what did we actually see, and when?"* The working state is the current picture: *"what do we think is true right now?"* Collapsing them means a revised belief silently rewrites the record that contradicted it — and then nothing can be reconstructed afterwards.

## Every fact carries provenance

A fact without provenance cannot be invalidated, because nothing knows what it depended on. The semantics each fact needs:

```json
{
  "claim": "auth suite passes",
  "source": "tool",                    // tool | repo | user | reasoner
  "observed_at": "<step or timestamp>",
  "scope": ["src/auth/**", "test/auth/**"],
  "invalidated_by": ["change within scope"]
}
```

A runtime need not store it this verbosely — but the four questions must be answerable: **what was claimed, who observed it, when, and what would make it untrue.**

Note the `source` values and what they are worth. `tool` and `repo` are observations. `user` is a requirement or constraint. `reasoner` is an *inference* — it is only as good as the facts it was drawn from, and it expires when any of them does. A conclusion is never stronger than its weakest input.

---

# 4. Updating

```text
ACTION → RESULT → UPDATE STATE → NEW DECISION
```

The update is mandatory before the next decision. A decision taken against stale state is a decision about a situation that no longer exists.

| On | Do |
|---|---|
| new observation | add to `facts` with its source; clear any `unknown` it answers; reset `steps_since_new_fact` |
| observation contradicts a hypothesis | remove the hypothesis; record what was actually observed; escalate to System 2 |
| new class-R trigger discovered | raise `craft.class`; **recompute autonomy from scratch** |
| action repeated | increment `same_action` |
| same error signature seen again | increment `same_failure` |
| no new fact this step | increment `steps_since_new_fact` |
| file edited | add to `changed_files`; set the affected `verification` entries back to `not_run` |

That last row matters more than it looks: **editing a file invalidates every check that ran before the edit.** A runtime that keeps `"tests": "passed"` across a subsequent edit is reporting evidence it no longer has.

## 4.1 Invalidation

Expiry is mechanical, not a judgement call. On every mutation, walk the state and expire anything whose scope the mutation touched. The matrix is the same one the gate enforces — `../03-GATE.md` §17.1 is authoritative; this is how a runtime applies it:

```text
mutation                      expires
─────────────────────────────────────────────────────────────
edit any file             →   facts sourced from that file
                              typecheck · lint · build · tests covering it
                              any reasoner conclusion drawn from those facts
                              "working tree clean"
dependency change         →   build · tests · security scan
config / env change       →   every check that read that config
branch or checkout change →   effectively all of it — rebuild state
```

Three rules keep this honest and cheap:

1. **Expire coarsely.** When it is unclear whether a fact survives, it does not. Re-observing is cheap; deciding on a stale fact is not. **Do not build dependency tracking** to keep a fact alive — that is a complicated way to be wrong occasionally.
2. **Expire, do not delete.** An expired fact becomes an `unknown`, which is what drives the next action (§3). Silently dropping it makes the gap invisible.
3. **Inferences expire with their inputs.** If a reasoner concluded "the client now sends the right token" from three facts and one of them expires, the conclusion expires too. Conclusions do not outlive their premises.

> The failure this prevents: test passes → agent makes one more small edit → state still says `tests: passed` → agent reports done. Nothing was fabricated. The result was simply describing code that no longer existed.

## Confidence does not carry over

Confidence attaches to a decision **taken against a particular state**. When the state changes materially, the previous confidence is void — not decayed, void. Recompute.

---

# 5. What must never enter state

| Never | Why |
|---|---|
| secrets, tokens, credentials, keys | state is sent to a provider and usually logged |
| personal or customer data | same, plus legal exposure |
| whole file contents | that is context, not state — reference the path |
| the transcript, or a summary of it | defeats the purpose of the layer |
| raw stack traces | keep the error signature, drop the frames |
| model-generated confidence from a previous turn | see §4 |

Redact at the point of construction, not at the point of logging. A redaction step that runs after the provider call has already leaked.

---

# 6. Keeping it small

| Field | Cap | Policy |
|---|---|---|
| `facts` | ~12 | keep those that still constrain the next decision; drop the settled and superseded |
| `unknowns` | ~6 | more than six means the task should be decomposed |
| `evidence` | ~15 | paths only, never contents |
| `last_result` | ~200 chars | the outcome, not the output |

Pruning is not forgetting: a dropped fact was one that no longer changes any answer. If pruning is difficult, the task is too large — decompose it rather than growing the payload.

---

# 7. State hash

Hash the canonical serialisation of state on every decision and store it in telemetry (`CALIBRATION.md` §2).

It gives three things for almost nothing:

- **loop detection that cannot be argued with** — an identical hash two decisions apart means no progress was made, whatever the agent believes;
- **replayability** — the same state can be replayed against a new provider or a new decision version;
- **honest calibration** — outcomes can be grouped by the state that actually produced them.
