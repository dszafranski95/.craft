# decide/CALIBRATION.md — earning the right to trust a number

**Version 2.2** · Implementation contract for the Fast Decision Layer. Concept and agent behaviour: `../05-DECIDE.md`.

> **Do not load this file for a normal coding task.** It specifies a runtime, not your behaviour.

---

# 1. The problem

A provider returns:

```text
edit 0.96
```

This does **not** mean there is a 96% chance that editing is the right move. It means the model's output distribution put 0.96 on the token mapped to `edit`. That is a statement about token preference, not about the world.

Language-model token probabilities are routinely overconfident, and the gap is not uniform — it varies by model, by decision, by prompt version, and by how the answer space was ordered. A raw logprob is a **signal with unknown units**.

Calibration is the process of converting that signal into something a policy can act on. Until it has been done:

> A threshold without calibration data is a guess with a decimal point.

Before measurement, every mutating decision goes through System 2 regardless of confidence (`POLICY.md` §3).

---

# 2. What to record

One record per decision. This is the raw material; without it, nothing below is possible.

```text
task_id
decision_id + version          route.next/v2
state_hash
provider + model + version
answer
distribution                   if available
policy_outcome                 acted | validated | escalated | asked_human | blocked
executed_action
action_result                  observed outcome, not intent
system2_override               did System 2 disagree?
human_override                 did a human disagree?
final_outcome                  did the task succeed, and was this step right in hindsight
latency_ms
tokens_in / tokens_out
```

**Never record:** secrets, credentials, personal data, or source contents. Record the state hash, not the state. Redact at construction (`STATE.md` §5).

Retention should be long enough to recalibrate after a model change — typically a few thousand decisions per decision id.

---

# 3. Getting ground truth

The hard part is not the maths, it is knowing what the right answer was. Three sources, in descending quality:

| Source | Quality | Cost |
|---|---|---|
| human label on a sampled decision | highest | high |
| System 2 verdict on an escalated decision | good | already paid for |
| outcome-derived label — did the step advance the task? | noisy but free | none |

Sample deliberately. Labelling every decision is unaffordable and unnecessary; a stratified sample across confidence bands is far more informative than a large sample of easy cases. **Deliberately over-sample the high-confidence band** — that is precisely the band the policy is about to trust, and the one where an error is least likely to be caught.

Outcome-derived labels have a known bias: an action that worked was not necessarily the *right* action. Treat them as a screening signal, not as a verdict.

---

# 4. What to measure

Per decision id, per version, per provider:

| Metric | Answers |
|---|---|
| accuracy | how often the chosen answer was right |
| precision / recall per answer | which specific answers are unreliable |
| confusion matrix | what gets mistaken for what — usually the most actionable output |
| Brier score | overall quality of the probabilities, not just the choice |
| expected calibration error (ECE) | how far stated confidence sits from observed accuracy |
| coverage vs accuracy | what fraction can be automated at a given accuracy |
| false-auto-action rate | how often autonomy was granted and was wrong — **the safety number** |
| human override rate | how often people disagreed with an executed decision |

The decisive one is the calibration curve: bucket decisions by stated confidence, then measure accuracy within each bucket.

```text
stated 0.90–1.00   →   observed 0.74     the 0.90 threshold is not usable
stated 0.90–1.00   →   observed 0.93     the 0.90 threshold is usable here
```

Set the threshold from the second column. Never from the first.

---

# 5. Setting a threshold

1. Pick the **acceptable false-auto-action rate** for the decision. This is a risk decision, not a statistical one — and it is lower for `route.next` leading to `edit` than for `route.next` leading to `inspect`, because the consequences differ.
2. Find the lowest confidence bucket whose observed accuracy meets it.
3. Set the threshold at that bucket's floor.
4. Record the sample size. A threshold from 40 observations is an anecdote.
5. Re-measure on any change to the model, the provider, the prompt prefix, the answer ordering, or the decision version.

Thresholds are **per decision, per version, per provider**. They do not transfer — not between models, not between versions of the same model.

## Coverage is a choice

Automation rate and accuracy trade against each other. Raising the threshold automates less and is right more often. The correct operating point depends on the cost of being wrong, which differs per decision — read-only steps can run at far lower confidence than anything that mutates.

---

# 6. Using telemetry honestly

| Use it for | Not for |
|---|---|
| setting and revising thresholds | proving the agent is good |
| finding decisions that are systematically wrong | a dashboard nobody acts on |
| detecting drift after a model change | justifying autonomy that was never measured |
| training a dedicated classifier | replacing tests |

**No measurement in this file is evidence about a code change.** Calibration measures the decision layer. Whether the code works is settled by `../03-GATE.md` and executed checks at E2/E3. A well-calibrated agent that shipped a broken change has a well-calibrated agent and a broken change.

---

# 7. Drift

Recalibrate whenever any of these changes:

- the model or its version — including a silent upgrade behind a hosted endpoint;
- the provider;
- the prompt prefix, the answer ordering, or the letter mapping;
- the decision version (`DECISIONS.md` §4);
- the state schema (`STATE.md` §2);
- the kind of work the agent does — thresholds learned on bug fixes do not hold for migrations.

Between recalibrations, monitor the two numbers that reveal drift earliest: **false-auto-action rate** and **human override rate**. A rise in either means the thresholds are already stale.

---

# 8. If none of this is measured

That is a legitimate position, and the common one. The honest configuration is then:

```text
conservative profile          no numeric thresholds
words, not numbers            clear | unsure | contested
mutations                     always through System 2
completion                    always through 03-GATE.md
deterministic rules           wherever the runtime can answer at all
```

This still removes most unnecessary deliberation, because the savings come mainly from **not calling the model** — for facts the runtime already has, for steps that are obviously read-only, for loops caught by counters — rather than from trusting a number.

> Measure before you automate. An uncalibrated threshold is not caution; it is a wrong number acted on automatically.
