# decide/STATE.md — the decision state

**Version 2.2** · Implementation contract for the Fast Decision Layer. Concept and agent behaviour: `../05-DECIDE.md`.

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
