# decide/POLICY.md — what the runtime is allowed to do with an answer

**Version 3.0** · Implementation contract for the Fast Decision Layer. Concept and agent behaviour: `../05-DECIDE.md`.

> **Do not load this file for a normal coding task.** It specifies a runtime, not your behaviour.

---

# 1. The separation

```text
the model     evaluates      "inspect, and I am fairly sure"
the policy    authorises     "inspect is permitted here; execute it"
```

The model never decides what it is allowed to do. That is not a stylistic preference — it is the only arrangement in which a wrong answer stays cheap.

Consequently:

| Property | Belongs to |
|---|---|
| which answer fits the state | model |
| whether that answer may execute | policy |
| whether a human is required | policy |
| whether the check actually passed | the exit code |

**Policy is code, not prompt.** An instruction like "please do not delete production data" is a request. This is a rule:

```ts
if (action.destructive) return requireHumanApproval(action)
```

Anything expressible as a deterministic rule must be a deterministic rule. Prompt text is what remains after that.

## The permission envelope

Policy authority and the current permission scope are two different gates, and both must open.

```text
provider says     edit_file, 0.99
policy says       edit is permitted for this risk band
permission says   this session has no write access
result            not executed
```

**A provider can never widen its own permissions.** If the runtime is read-only, `edit_file` is not a slow path or a confirmation prompt — it is not an available answer at all (`DECISIONS.md` D-02). The answer space is built from what this session can actually do, so an unauthorised action is unrepresentable rather than merely refused.

---

# 2. Evaluation order

Evaluate top-down; **stop at the first rule that matches**. Order is part of the contract — reordering these changes the safety properties.

```text
 1. craft.stop                        → human. No exception, no override.
 2. risk = critical                   → human. No confidence unlocks this.
 3. provider failed / unavailable     → decision default (DECISIONS.md), no autonomy
 4. same_action > 2                   → System 2 replan
 5. same_failure >= 2                 → System 2 replan
 6. steps_since_new_fact >= 5         → System 2 replan
 7. evidence contradicts current plan → System 2
 8. risk = high                       → System 2 designs; mutation needs its approval
 9. reasoning.required above threshold→ System 2
10. confidence below act threshold    → inspect further, or System 2
11. confidence in the validate band   → System 2 validates, then act
12. otherwise                         → direct action permitted
```

Rules 1–3 are **absolute**: they are evaluated before any confidence value is read. A provider outage must degrade the system toward caution, never toward autonomy.

---

# 3. Thresholds

Thresholds are only meaningful once measured (`CALIBRATION.md`). Until then use the conservative starting points below and treat them as placeholders, not settings.

```text
reasoning.required   >= 0.70   → System 2
direct action        >= 0.90   AND risk <= moderate AND Craft permits
validate band        0.70 – 0.90 → System 2 validates before acting
below                <  0.70   → gather more evidence, or System 2
```

Three conditions on their use:

1. **A threshold without calibration data is a guess with a decimal point.** Before measurement, run every mutating decision through System 2 regardless of confidence.
2. **A provider without logprobs has no confidence.** It runs at Tier B (`PROVIDERS.md` §2) — not an invented number.
3. **Thresholds are per decision, per version, per provider.** `route.next/v1` at 0.90 on one model says nothing about `route.next/v1` on another.

## Policy is versioned too

The rules and thresholds in this file form a version — `policy/v1` — and every telemetry record stores it (`CALIBRATION.md` §2). Changing a threshold, reordering §2, or altering what a band permits produces `policy/v2`.

Without it, a measured change in outcomes cannot be attributed: you will not know whether the agent got worse because the model changed, the decision criteria changed, or someone moved a threshold by 0.05.

---

# 4. What confidence may never unlock

No value of any signal permits:

- passing a Craft STOP condition;
- an irreversible operation without human approval — migration, deletion, force-push, key rotation, published-contract break;
- committing, pushing or deploying without explicit instruction;
- suppressing a compiler, type, lint or security check;
- weakening, skipping or deleting a test;
- declaring completion without the gate;
- recording a check as passed when it was not executed.

These are not thresholds set very high. They are **not inputs to the decision system at all**. A runtime that implements them as "requires confidence ≥ 0.99" has implemented them wrongly.

---

# 5. Before a mutation: the action envelope

`edit` is not an executable instruction. Before anything writes, the runtime must hold a bounded description of what is about to happen:

```json
{
  "action": "edit_file",
  "scope": ["src/auth/session.ts"],
  "intent": "make the refreshed token canonical for the API client",
  "expected_effect": "client sends the current session token on every request",
  "verification": ["auth regression test", "typecheck"],
  "risk": "moderate",
  "reversible": true
}
```

The fast provider does not produce this. It contributes at most `action`; the rest comes from deterministic classification, from System 2's design, or from the task contract (`../02-PROTOCOL.md` A1). The point is not the JSON — it is that **five questions have answers before a write happens**:

```text
what · where · why · what should change · how it will be checked
```

Three rules:

1. **Scope is declared and enforced.** A write outside the declared scope is a policy violation, not a surprise. This is what stops an "edit one file" decision from becoming a nine-file refactor.
2. **`verification` is chosen before the edit, not after.** Choosing the check after seeing the result is how you end up picking the check that passes.
3. **An envelope that cannot be filled is not ready to execute.** If `expected_effect` cannot be stated, the change is not understood — the answer is `inspect`, not `edit` (`DECISIONS.md` D-05).

This is deliberately the same discipline Craft already requires of a human making a change. The envelope only makes it machine-checkable.

---

# 6. Deterministic first

Before any provider call, answer everything the runtime can answer itself:

| Question | Determined by |
|---|---|
| is this tool available? | the tool registry |
| is this operation destructive? | a static classification of the operation |
| does this path exist? | the filesystem |
| did the check pass? | the exit code |
| is this path protected? | a glob list in configuration |
| is this class R? | the trigger list in `../00-START.md` §3 Step 2 |
| have we seen this state before? | the state hash |

Every question answered here is a provider call not made, a token not spent, and a wrong answer not possible. This is where most of the layer's savings actually come from — not from the classifier being cheap, but from never calling it.

---

# 7. Escalation contract

System 2 receives the state, the decision that triggered escalation, and the reason. It returns a patch, not a narrative:

```json
{
  "finding": "token is written to session store but the API client reads a cached header",
  "proposed_next_action": "inspect the session → api-client synchronisation boundary",
  "state_patch": {
    "facts_add": [{ "claim": "api client caches header at construction", "source": "api/client.ts:41" }],
    "unknowns_clear": ["whether the refreshed token reaches the API client"],
    "constraints_add": ["token ownership must stay in auth/session.ts"]
  }
}
```

Rules:

- System 2 **may** overrule any fast decision; the reverse is never true.
- System 2 does not restart the task. It answers the question that caused the escalation.
- Its output is merged into state and the loop resumes. It does not become a second agent with its own memory.
- If System 2 also cannot resolve it, the next step is `ask_user` — not another attempt.

---

# 8. Failure modes and their required behaviour

| Failure | Required behaviour |
|---|---|
| provider timeout or error | use the decision default; do not retry silently more than once |
| provider returns an answer outside the space | discard, use default, count as a provider failure |
| provider unavailable repeatedly | disable the layer; run on System 2 alone. **The agent must still work with no fast layer at all** |
| state exceeds the size cap | prune per `STATE.md` §6; if it cannot be pruned, decompose the task |
| two decisions contradict each other | System 2; never reconcile them by averaging |
| the same state hash recurs | loop; force replan regardless of any counter |

The last-resort property of the whole design: **removing the fast layer must degrade cost, never correctness.** If disabling it breaks the agent, the layer has been given authority it was never meant to hold.
