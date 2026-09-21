# decide/PROVIDERS.md — the provider interface

**Version 2.2** · Implementation contract for the Fast Decision Layer. Concept and agent behaviour: `../05-DECIDE.md`.

> **Do not load this file for a normal coding task.** It specifies a runtime, not your behaviour.

---

# 1. The contract

Craft defines the Fast Decision Layer. Providers merely implement it. Nothing above this file may name a vendor, a model, an API or a hosting arrangement.

```text
FastDecisionProvider

  decide(decision, state, options) -> {
    answer        one member of decision.answers
    distribution  optional: answer -> weight, only if the backend really produces one
    confidence    optional: present only when distribution is present
    provider      identifier, for telemetry
    model         identifier and version, for telemetry
    latency_ms
  }

  capabilities() -> {
    constrained_output   bool    can the answer space be enforced?
    logprobs             bool    are token probabilities available?
    batch                bool    can independent decisions share one call?
  }
```

Implementations may include a dedicated classifier, a local model with constrained decoding, a hosted model with structured output, or a rules engine. The runtime must not care which. Swapping providers is a configuration change; it must never be a redesign.

A provider that cannot answer returns nothing and the runtime applies the decision default (`DECISIONS.md` §1). It must never return a best guess dressed as an answer.

---

# 2. Two capability profiles

| | Full | Conservative |
|---|---|---|
| Requires | constrained output **and** logprobs | structured output only |
| Returns | answer + distribution | answer only |
| Numeric thresholds | permitted, after calibration | **forbidden** |
| Direct action on confidence | permitted within `POLICY.md` | not permitted |
| Mutating decisions | per policy | always via System 2 |

The conservative profile is fully supported and is the correct default. A provider with no distribution still eliminates most unnecessary deliberation — which is where the savings are — it simply may not claim a probability it does not have.

Degrading to conservative must be automatic when `capabilities().logprobs` is false. It must never be a setting an operator can wrongly turn on.

---

# 3. Building a provider from an ordinary LLM

Any model that supports constrained decoding or grammar-restricted output can serve as a fast decision provider. Map the answer space onto single tokens:

```text
A = inspect        E = ask_user
B = edit           F = human_review
C = test           G = done_candidate
D = research
```

Request:

```text
<state, per STATE.md>

Return exactly one token: A B C D E F G
```

Settings:

```text
reasoning / thinking   off
max_tokens             1
constrained output     on
temperature            0
logprobs               on if supported
```

With logprobs, one forward pass yields a distribution:

```text
A 0.92   B 0.03   C 0.03   D 0.01   ...
```

The runtime maps `A → inspect` and hands the pair to `POLICY.md`. Without logprobs, the same call still yields a valid answer — just no confidence.

## Why single tokens

One output token is the whole point. It is the difference between a decision costing one token and a decision costing a paragraph of reasoning that nobody reads. The letters exist only because they are reliably single tokens; the model never sees them as meaningful labels, so the mapping must live in the runtime, not in the prompt.

## Non-negotiables

- **Letters are positional and must be stable within a decision version.** Reordering the mapping invalidates calibration as surely as changing the criteria (`DECISIONS.md` §4).
- **Never allow free text.** Unconstrained output reintroduces parsing, latency and the failure modes this design exists to remove.
- **Never enable thinking on a fast decision.** A reasoning trace before a one-token answer costs more than the System 2 call it was meant to avoid.

---

# 4. Prompt economy

The fast prompt is a hot path: it runs many times per task. Treat every token in it as recurring cost.

| Do | Do not |
|---|---|
| put the fixed part — criteria, answer space, format — in a stable prefix, so it caches | rebuild the prompt string each call |
| vary only the state at the tail | resend instructions that never change |
| keep total input in the 200–1,500 token range | attach transcript, file contents or history |
| use the same prefix bytes for the same decision version | interpolate timestamps or ids into the prefix |

A stable prefix with a varying tail is what makes prompt caching work. Getting this wrong is the most common reason a "cheap" layer turns out not to be cheap.

---

# 5. Batching

If `capabilities().batch` is true, independent decisions may share one call — for example `route.next`, `reasoning.required`, `risk.operational` and `tool.next` against the same state.

Two constraints:

1. **Answers stay separate.** One call, four answers, four distributions. Never one fused verdict (`DECISIONS.md` §2).
2. **Only genuinely independent decisions batch.** `tool.next` depends on `route.next`; batching them means the tool was chosen without knowing the step. Either resolve the dependency first, or restrict `tool.next` to the intersection of tools valid for every plausible route.

---

# 6. Choosing a provider

| Situation | Reasonable choice |
|---|---|
| starting out, no infrastructure | none — the agent runs the loop itself (`../05-DECIDE.md`) |
| local model already running | that model, constrained output, conservative profile |
| cost-sensitive, high step count | smallest capable model with logprobs, full profile after calibration |
| decisions are the bottleneck | dedicated classifier trained on telemetry (`CALIBRATION.md`) |
| regulated or audited environment | deterministic rules wherever possible; the model answers only residual cases |

Start at the top. The first row is not a placeholder — an agent that merely refuses to deliberate before a `grep` has already captured most of the benefit, at zero infrastructure cost.

---

# 7. Provider independence in practice

Three properties make provider swaps cheap. Verify them deliberately:

1. **No provider name appears above `decide/`.** `00`–`05` describe a contract only.
2. **Telemetry records provider and model on every decision** (`CALIBRATION.md` §2). Without it, a regression after a swap is undiagnosable.
3. **Calibration is per provider.** Thresholds do not transfer between models — not even between versions of the same model. On a swap, the previous profile applies until the new one is measured.

The design goal is blunt: **replacing the provider in a year must be a configuration change, not a redesign.**
