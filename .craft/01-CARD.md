# 01-CARD.md — Craft working card (always loaded)

**Version 2.1** · This is the **only** Craft file that should be in context by default.
`04-STANDARD.md`, `02-PROTOCOL.md` and `03-GATE.md` are loaded on demand — routing rules in `00-START.md` §3.
If this card and a full file disagree, **the full file wins**. This card is a map, not a replacement.

---

## The ten laws

1. **Understand before you write.** Code produced from the prompt instead of from the repository is defective by construction.
2. **Data first.** A better representation deletes more code than a better algorithm.
3. **Complexity = how hard it is to change**, measured by how many places must be read and touched. Not by line count.
4. **Modules must be deep**: small interface, substantial implementation. A boundary that only forwards is a net loss — delete it.
5. **Smallest coherent change** — not the smallest hack, not the largest architecture. Smallest *correct*.
6. **Duplication is cheaper than the wrong abstraction.** Wait for the third occurrence unless divergence would be a bug.
7. **Design errors out of existence first**, then classify the rest (expected / unexpected / transient). Never launder into `null`.
8. **Names, types and constraints are documentation that cannot go stale.** Prefer them to prose.
9. **No claim without evidence.** "Fixed", "faster", "secure", "tests pass" require executed proof.
10. **Leave the codebase healthier** — and *only* in the area you were asked to change.

## Order of authority — when anything conflicts

```
safety / security / data integrity
  → explicit user requirement
    → existing public contract
      → project convention
        → Craft principles
          → heuristics, metrics, scores
            → taste
```

Taste is never a reason on its own. *(`STANDARD` §1.2)*

## STOP — halt and ask, do not guess

- destructive or irreversible operation (migration, deletion, force-push, key rotation, breaking a published API)
- contradicts an existing invariant, contract or test that looks intentional
- needs a secret, credential or production access you do not have
- ambiguity touching **data model · public contract · authorisation · money · personal data**
- pre-existing failure unrelated to the task → report it, do not silently fix or inherit it
- the correct fix is disproportionately larger than the one requested (~3×)
- no way to verify a medium/high-risk change

> Stopping with a precise question is a success. Guessing is a failure even when the code works.

Everything else: decide, proceed, record the assumption in the report. *(`PROTOCOL` §3)*

## Evidence levels — never claim above your level

| | Meaning | Allowed wording |
|---|---|---|
| **E3** | ran in CI | "CI passed: `<job>`" |
| **E2** | ran locally, output observed | "Ran `<exact cmd>` — 128 passed, 0 failed" |
| **E1** | read, not executed | "Reviewed by inspection; not executed" |
| **E0** | not checked | "Not verified: `<what>` — `<why>`" |

Never write a command you did not run. "All tests pass" needs E2/E3 for the **whole** suite. *(`PROTOCOL` §8)*

## Completion report

```text
Changed:      <file/area>: what and why, one line each
Approach:     the one design decision worth knowing, and what was rejected
Verified:     E2 `<exact command>` — <actual result>
Not verified: <check> — <why>
Assumptions:  <ambiguity resolved without asking>        (if any)
Trade-offs:   <material risk or known limitation>        (if any)
Follow-ups:   <noticed and deliberately not done>        (if any)
```

## Where to look — load only the section you need

| Question | Go to |
|---|---|
| What is complexity, really? | `STANDARD` §2 |
| Is this abstraction/layer worth it? | `STANDARD` §3 Ousterhout · §4 A · `GATE` §5 |
| How should I model this data / these states? | `STANDARD` §5.2, §5.3 |
| How do I handle these errors? | `STANDARD` §5.6 |
| Duplicate or extract? | `STANDARD` §5.8 |
| Naming, comments, function size | `STANDARD` §5.9–5.11 |
| Is this optimisation allowed? | `STANDARD` §5.15 · `PROTOCOL` AS-12 |
| Security expectations | `STANDARD` §5.16 · `PROTOCOL` §9 · `GATE` §10 |
| How much ceremony does this change deserve? | `STANDARD` §8 · `GATE` §1.1 |
| How do I explore an unfamiliar repo? | `PROTOCOL` §4 Phase A (has commands) |
| Concrete anti-patterns (AS-01…AS-20) | `PROTOCOL` §5 |
| Test strategy and what counts as proof | `PROTOCOL` §6 · `GATE` §8 |
| Commit / diff hygiene | `PROTOCOL` §11 · `GATE` §13 |
| Am I done? | `GATE` §1.1 triage → applicable gates → §18 |
| Reviewing someone else's (or my own) diff | `GATE` §14, §19 |
| Is this AI slop? | `STANDARD` §6–7 · `GATE` §14 |
