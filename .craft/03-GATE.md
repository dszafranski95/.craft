# 03-GATE.md — Craft Verification Gate

**Version 2.2**

> A change is complete only when it passes every **applicable** BLOCKER gate.
>
> Scores are diagnostic. A high score never overrides a blocker.
> Conflicts are resolved by `04-STANDARD.md` §1.2.

**When to load this file** *(routing: `00-START.md` §3)* — before declaring any class S or R change complete, and whenever reviewing a diff. Class S: read §1.1, then only the gates it selects, then §19. Class R: the whole file, plus the independent review prompt in §20. Not needed while exploring or designing.

---

# 1. Gate model

Severity:

- **BLOCKER** — cannot be accepted without explicit human risk acceptance.
- **REQUIRED** — expected for normal production changes; omission needs a documented reason.
- **ADVISORY** — a heuristic that triggers review, never automatic rejection.

The gate is technology-neutral. Map commands to the repository's stack — and to its CI configuration, which is the real definition of "passing".

## 1.1 Triage — which gates apply

Running twelve gates on a typo is ceremony, and ceremony is waste. Classify the change first, then apply the matching depth. **Class is decided by blast radius and reversibility, not by diff size.**

| Class | Examples | Gates applied |
|---|---|---|
| **T — Trivial** | comment, copy string, formatting confined to touched lines, obvious dead-code removal | Gate 0, Gate 5 (checkers pass), Gate 11 |
| **S — Standard** | business logic, new endpoint, bug fix, compatible schema addition, refactoring | Gates 0–7, 11, 12, 13 |
| **R — Risky** | auth/authz, payments, personal data, migrations, concurrency, cryptography, deletion, wire/public API contract, multi-tenant boundary, hot path | **All gates**, plus rollback plan, plus human review — never auto-accepted |

If unsure between two classes, choose the higher one.

**One-way doors** (irreversible: migrations, deletions, published contracts, key rotation) are always class R regardless of size.

---

# 2. Gate 0 — Scope and understanding

## BLOCKER

- [ ] The requested behaviour is understood.
- [ ] No material requirement was invented.
- [ ] Relevant existing implementation was inspected.
- [ ] Relevant canonical data/types were identified.
- [ ] Relevant existing tests were inspected.
- [ ] The change is in the correct architectural area.
- [ ] No STOP condition (`02-PROTOCOL.md` §3) was passed over silently.

> Fail this gate if the implementation was produced primarily from the prompt without understanding the repository.

---

# 3. Gate 1 — Correctness

## BLOCKER

- [ ] The intended behaviour works.
- [ ] Important failure paths have deliberate behaviour.
- [ ] Data invariants remain valid.
- [ ] Public contracts remain compatible unless intentionally changed.
- [ ] A bug fix has a regression proof that fails before the fix.
- [ ] Concurrency/async behaviour is reasoned about where applicable.

Ask only the questions relevant to this change:

empty input? · invalid input? · duplicate input? · boundary values (0, 1, max, negative, unicode)? · timeout? · partial external failure? · retry? · repeated request (idempotency)? · stale state? · simultaneous requests? · cancellation? · clock skew / timezone / DST? · very large input?

---

# 4. Gate 2 — Data and state design

## BLOCKER

- [ ] There is no unnecessary second source of truth.
- [ ] State ownership is clear.
- [ ] Important states are represented explicitly.
- [ ] State transitions preserve invariants.
- [ ] Persistence constraints match application assumptions.
- [ ] Illegal states are unrepresentable where the language allows it.

## REQUIRED

- [ ] Derived data is derived, not independently synchronised.
- [ ] Untrusted input is **parsed** into a domain type at the boundary, not merely validated and passed on.
- [ ] Cross-boundary representations are converted in one intentional place.

**Red flag:**

> A large amount of code exists only to keep two representations synchronised.

Revisit the data model — do not improve the synchronisation code.

---

# 5. Gate 3 — Simplicity and abstraction

## BLOCKER

- [ ] No speculative feature or generalisation was added.
- [ ] No obviously unnecessary layer was introduced.
- [ ] No false abstraction created solely to eliminate visual duplication.
- [ ] No complex workaround masks a simpler representation.
- [ ] **No shallow module**: every new boundary hides more than its interface costs.

## REQUIRED

- [ ] Each new abstraction has a one-sentence responsibility with no "and".
- [ ] Each new dependency has a concrete justification.
- [ ] Control flow can be followed without excessive jumping.
- [ ] Nested complexity is reasonable for the task.
- [ ] Deleting any single new element would make the change worse, not better.

## Deep-module check

For each new class, module, layer or wrapper:

```text
What does a caller need to know to use it?      (interface cost)
What does it save the caller from knowing?      (hidden benefit)
```

If cost ≥ benefit — pass-through methods, forwarding wrappers, a layer whose interface mirrors the layer beneath — **delete the boundary**.

## ADVISORY smells and soft thresholds

Numbers below are **inspection triggers**, never automatic rejections, and never a reason to split working code that is clear (`02-PROTOCOL.md` AS-13).

| Signal | Inspect when roughly |
|---|---|
| nesting depth | > 3 |
| function parameters | > 4, or any 2+ booleans |
| behaviour-switching boolean flags | ≥ 1 |
| function length | > 60 lines *and* multiple abstraction levels |
| file/class length | > 500 lines, or > 1 reason to change |
| cyclomatic / cognitive complexity | > 10 / > 15 |
| new files added for one feature | > 3 |
| layers between caller and effect | > 3 |
| public methods on a new class | > 7 |
| indirection to read one behaviour end-to-end | > 3 hops |

Also inspect: repeated branching on the same enum in many places · broad `utils` · many wrappers · duplicated code · deep inheritance · shotgun surgery · feature envy.

> A smell means "inspect the design", not "apply an automatic refactor".

---

# 6. Gate 4 — Readability and idiomaticity

## BLOCKER

- [ ] Comprehensible to a competent developer in this ecosystem on one reading.
- [ ] Names do not materially misrepresent behaviour.
- [ ] No intentionally clever construct hides important semantics.

## REQUIRED

- [ ] Domain vocabulary is used; the same concept always uses the same word.
- [ ] The code follows local style.
- [ ] Language-native idioms preferred over transplanted patterns.
- [ ] Comments explain non-obvious reasoning, not syntax.
- [ ] No comment was made untrue by this change.
- [ ] Public APIs are documented to project standards.
- [ ] No debug output, commented-out code, or unowned `TODO` remains.

**Rule:**

> Existing local convention beats a model's generic preference — unless the convention is itself unsafe, or is what is being changed.

---

# 7. Gate 5 — Type and static correctness

## BLOCKER (where applicable)

- [ ] Compiler passes.
- [ ] Type checker passes.
- [ ] No new unjustified suppressions.
- [ ] No new high-severity static-analysis findings.

Reject shortcuts such as:

```text
any    ignore    nolint    unchecked cast    unsafe    disable warning    eslint-disable
```

when their purpose is only to silence a design or correctness problem.

A suppression is acceptable only when **all four** hold:

1. the checker genuinely cannot model valid behaviour,
2. the scope is as narrow as the language allows,
3. the reason is documented inline,
4. a safer alternative was considered and rejected.

---

# 8. Gate 6 — Tests

## BLOCKER

- [ ] Relevant existing tests pass — with observed output, not assumption.
- [ ] New/changed behaviour has appropriate verification.
- [ ] No test was weakened merely to accept unintended output.
- [ ] No failing test was silently skipped, deleted or marked expected-to-fail.
- [ ] **Each new test was observed to fail against broken behaviour** (or is a regression test that failed before the fix).

## REQUIRED

- [ ] Tests assert behaviour and invariants, not implementation structure.
- [ ] Tests are deterministic: injected clock, seeded randomness, no real network, no sleeps, no inter-test order dependence.
- [ ] Test names explain the behaviour under test.
- [ ] Important negative paths are covered.
- [ ] Mocks do not stand in for the behaviour being proven.
- [ ] Test level matches risk; the cheapest sufficient level was chosen.
- [ ] No logic (loops/conditionals computing expectations) inside tests.

**Evidence strength, weakest to strongest:**

```text
coverage %  <  test exists  <  test observed red then green  <  mutation score  <  property-based invariant
```

Coverage is evidence of execution, not of correctness. A numeric threshold may supplement this gate; it never replaces review.

---

# 9. Gate 7 — Error handling and observability

## BLOCKER

- [ ] Meaningful failures are not silently swallowed.
- [ ] Error propagation preserves enough context to diagnose the issue.
- [ ] Sensitive information is not exposed in errors or logs.
- [ ] Retry behaviour is safe (bounded, backed off, idempotent) where present.
- [ ] Expected and unexpected failures are distinguished; unexpected failures fail loudly.

## REQUIRED for production paths

- [ ] Important failures are observable.
- [ ] Logging/metrics/tracing match local convention.
- [ ] Logs are actionable rather than noisy; no high-cardinality field is logged carelessly.
- [ ] No success is logged before success is known.
- [ ] Errors that cross a boundary carry a correlation identifier where the system uses one.

**Ask:** if this fails at 3 a.m., can someone tell *what* failed, *for whom*, and *whether to retry* — from the logs alone?

---

# 10. Gate 8 — Security

Apply when the code crosses a trust boundary or handles sensitive capability or data.

## BLOCKER

- [ ] Authentication behaviour is correct where relevant.
- [ ] Authorisation is enforced server-side, at the trusted boundary, on every path.
- [ ] Untrusted input is validated/parsed appropriately at the edge.
- [ ] Secrets are not committed, logged, or embedded in errors, tests or fixtures.
- [ ] No known dangerous primitive is introduced unnecessarily.
- [ ] New high/critical vulnerability findings are resolved.
- [ ] File/path/URL/query construction is safe for its context (no injection, no traversal, no SSRF).
- [ ] No custom cryptography; established platform primitives only.

## REQUIRED

Use OWASP ASVS or the relevant platform guidance for security-sensitive applications.

Raise review intensity for: auth · payments · personal/sensitive data · privileged operations · cryptography · uploads · remote fetches · parsers · deserialisation · multi-tenant boundaries.

---

# 11. Gate 9 — Performance and resource use

## BLOCKER

- [ ] No unsupported claim of performance improvement is used to justify added complexity.
- [ ] No obvious unbounded behaviour where inputs can grow (unbounded query, unbounded buffer, unbounded concurrency, unbounded retry).
- [ ] No accidental N+1, repeated remote call, or pathological loop in a material path.
- [ ] Resources are released deterministically (connections, files, handles, subscriptions, timers).

## REQUIRED when performance is a requirement

- [ ] Workload defined · baseline measured · bottleneck identified · new result measured.
- [ ] Memory/resource trade-off considered.
- [ ] Benchmark reproducible enough to be useful (environment recorded).

> Measure before tuning. But choosing the right data structure is design, not tuning — do that up front.

---

# 12. Gate 10 — Dependency discipline

## BLOCKER

- [ ] No dependency added for functionality already available in the project or platform.
- [ ] No duplicate library category introduced without reason.
- [ ] No obvious security, licensing or maintenance problem.
- [ ] Lockfile/manifest changes are intentional and confined to this change.

## REQUIRED

For each new dependency, document briefly:

```text
Need:
Existing alternatives considered:
Why this dependency:
Operational/security cost:
Licence:
```

Tiny convenience is not worth permanent dependency surface.

---

# 13. Gate 11 — Change hygiene

## BLOCKER

- [ ] No unrelated behaviour change hidden in the diff.
- [ ] Generated artifacts, secrets or debug files are not included.
- [ ] The repository is not left in a broken intermediate state.

## REQUIRED

- [ ] The diff is focused; every changed file is necessary.
- [ ] Large refactoring is separated from feature work when practical.
- [ ] Migration and rollback considerations exist where relevant.
- [ ] Documentation, config examples and contracts are updated where behaviour changed — and nowhere else.
- [ ] Pre-existing problems found along the way are **reported, not silently fixed and not silently inherited**.

Google's small-change principle applies conceptually: *small* means one coherent idea, not a line count.

---

# 14. Gate 12 — AI-slop inspection

Reject or revise if the diff shows a pattern of:

- [ ] duplicated existing project functionality,
- [ ] generic `manager`/`helper`/`service` layers without distinct semantics,
- [ ] speculative extension points,
- [ ] many caller-specific flags in one shared abstraction,
- [ ] repeated broad casts or ignored types,
- [ ] catch-and-continue error laundering,
- [ ] defensive checks for impossible states,
- [ ] comments narrating syntax; docstrings restating signatures,
- [ ] tests mirroring mocks rather than behaviour,
- [ ] a new package for trivial functionality,
- [ ] multiple new files where one direct change would suffice,
- [ ] brute-force branches added until a test happened to pass,
- [ ] architecture copied from another ecosystem rather than local idioms,
- [ ] configuration, flags or env vars nobody requested,
- [ ] README/doc inflation, marketing register, emoji, unbenchmarked superlatives,
- [ ] placeholders, stubs or hardcoded sample data presented as complete,
- [ ] completion claims unsupported by executed checks.

One item may be legitimate. **A cluster is a strong slop signal** — and the correct response is to re-read the repository, not to patch the symptoms.

---

# 15. Gate 13 — Agent autonomy and decision integrity

Applies when an agent decided its own next steps — which, for any AI-assisted change, is always. It audits **how the work was driven**, not what the code does; the other gates cover the code. Definitions: `05-DECIDE.md`.

## BLOCKER

- [ ] No Craft STOP condition was routed past. Confidence is not an override.
- [ ] No destructive or irreversible operation was performed on the agent's own authority.
- [ ] No risky mutation rests on a fast decision alone — class R changes were reasoned through and, where required, reviewed.
- [ ] Every decision was taken against **current** state: no conclusion carried over a material change, no check reported as passing after a later edit invalidated it.
- [ ] A repeated action or a repeated identical failure was escalated and replanned, not retried.
- [ ] No hypothesis is presented as an observed fact.
- [ ] No test, typecheck or build status was recorded from expectation rather than from an executed result (`02-PROTOCOL.md` §8).
- [ ] Model confidence appears nowhere as evidence of correctness.

## REQUIRED

- [ ] The work stopped and asked where the honest answer was "I cannot resolve this", rather than producing a plausible guess.
- [ ] Assumptions made in place of missing facts are in the report, not left implicit.
- [ ] Recon was proportional: enough to name the canonical owner of the changed data, not a sweep of the repository.

> The failure this gate catches is specific: **the work looks complete because the agent was confident, not because anything was verified.** If any claim in the report traces back to a decision rather than to an executed check, it fails here.

---

# 16. Suggested automated pipeline

Use the equivalents the repository actually defines in CI.

```text
1. format check
2. lint
3. compile / typecheck
4. unit tests
5. integration tests
6. build / package
7. static analysis
8. dependency / security scan
9. end-to-end tests where appropriate
10. benchmark / load test where required
```

Order may be optimised for fast failure. Do not make slow, expensive checks mandatory for a trivial local edit if CI provides them later — **but never claim they passed until they actually ran** (`02-PROTOCOL.md` §8).

---

# 17. Evidence table

Attach to any non-trivial change. This is the honest core of the gate.

| Check | Command | Level | Result |
|---|---|---|---|
| typecheck | `<exact command>` | E2 | passed |
| unit tests | `<exact command>` | E2 | 128 passed, 0 failed |
| integration | — | E0 | not run: no database in this environment |
| security scan | `<exact command>` | E3 | CI job `security` passed |

Levels: **E3** CI · **E2** executed locally, output observed · **E1** inspected by reading · **E0** not checked.

Rule: **no claim above its evidence level.** An empty row is honest; a fabricated one is disqualifying.

---

# 18. Craft score

Diagnostic only, for review conversations. Never a substitute for blockers, and never a target — a score optimised for is a score destroyed.

Rate each 0–5:

| Dimension | 0 | 5 |
|---|---|---|
| Correctness | unknown/broken | behaviour and edge cases well evidenced |
| Data design | duplicated/unclear state | canonical, explicit, coherent |
| Simplicity | tangled/over-engineered | minimal conceptual complexity |
| Readability | hard to follow | locally understandable and idiomatic |
| Abstraction depth | shallow/arbitrary boundaries | stable, deep semantic boundaries |
| Test quality | theatre/missing | behaviour-focused, risk-appropriate, proven able to fail |
| Reliability | hidden failures | deliberate failure behaviour |
| Security | unreviewed risk | risk-appropriate controls verified |
| Performance | guesses/pathologies | measured where material |
| Maintainability | change causes ripples | change remains localised |
| Operability | opaque | diagnosable in production where needed |
| Scope discipline | polluted diff | one coherent change |

Maximum: 60.

- **54–60** — exemplary, if no blockers.
- **48–53** — strong production code, if no blockers.
- **42–47** — acceptable only after reviewing the weak dimensions.
- **< 42** — likely needs redesign.

Self-assessment by the author of the change is the weakest form of this score. Treat it as a prompt for conversation, not as a result.

> **59/60 with a security blocker still fails.**

---

# 19. Definition of Done

A change is "Craft Complete" only when:

```text
[ ] requirement satisfied, nothing invented
[ ] relevant repo context inspected
[ ] data model and invariants understood
[ ] implementation is the smallest coherent solution
[ ] no unjustified abstraction, no shallow boundary
[ ] no unjustified dependency
[ ] no hidden error suppression, no silenced checker
[ ] new tests were observed failing before they passed
[ ] relevant tests pass (E2/E3, output observed)
[ ] compiler / type / lint / static checks pass as applicable
[ ] security reviewed proportionally to risk
[ ] performance measured if performance was a requirement
[ ] diff reviewed line by line
[ ] documentation and contracts updated where behaviour changed
[ ] evidence table filled honestly, including E0 rows
[ ] no claim in the report traces back to a decision rather than an executed check (Gate 13)
[ ] remaining uncertainty, assumptions and follow-ups stated explicitly
```

---

# 20. Independent review prompt

For a second pass — human or a separate model instance — with no memory of writing the code:

```text
Review this diff against .craft/04-STANDARD.md and .craft/03-GATE.md.
You did not write it. Do not assume it is correct.

1. Restate what this change actually does, from the diff alone.
2. Name every assumption it makes that the diff does not prove.
3. List the failure modes not handled.
4. Identify each new abstraction and say whether it hides more than it exposes.
5. Find the shallowest boundary in the diff and argue for deleting it.
6. Name anything that would break if this shipped and one assumption were wrong.
7. Which BLOCKER gates fail? Quote the specific line for each.
8. What in this diff would a competent reviewer ask to be deleted?

Do not praise. Report only what is wrong, missing or unproven.
```

The value of this pass comes from the reviewer not being the author. Do not run it in the same context that produced the code if avoidable.

---

# 21. Merge question

The final question is not:

> "Does this look clean?"

It is:

> "Do we have enough reason and evidence to believe this change is correct, understandable, appropriately simple, safe to operate, and leaves the codebase at least as healthy as it was before?"

If not, it is not ready.
