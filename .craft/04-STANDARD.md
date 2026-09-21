# 04-STANDARD.md — The Craft Code Standard

**Version 2.2** — universal engineering standard for human- and AI-written production code.

> This document defines **what good code is** and **why**.
> `02-PROTOCOL.md` defines **how to work**. `03-GATE.md` defines **how to prove it**. `01-CARD.md` is the always-loaded summary.
> Language- and framework-agnostic. Project rules may add strictness; they must never silently weaken this standard.

**When to load this file** *(routing: `00-START.md` §3)* — class R changes, design and architecture decisions, review and audit, or when a rule conflict needs the authority in §1.2. For everything else, read the single section the card's index points to. Section §0 below is the summary; if you only need that, you already have it on the card.

---

# 0. One page

If nothing else is read, these ten laws apply.

1. **Understand before you write.** Code produced from the prompt instead of from the repository is defective by construction.
2. **Data first.** A better representation deletes more code than a better algorithm.
3. **Complexity is what makes change hard** — measured by how many places a person must read and touch, not by line count.
4. **Modules must be deep**: small interface, substantial implementation. A layer that only forwards is a net loss.
5. **The smallest coherent solution wins**, and "coherent" forbids both over-engineering and hacks.
6. **Duplication is cheaper than the wrong abstraction.** Wait for the third occurrence and for the concept to stabilise.
7. **Design errors out of existence first**, handle the rest explicitly, never launder them into `null`.
8. **Names, types and constraints are documentation that cannot go stale** — prefer them to prose.
9. **No claim without evidence.** "Fixed", "faster", "secure", "tests pass" require executed proof.
10. **Leave the codebase healthier than you found it** — and *only* in the area you were asked to change.

---

# 1. Core definition

**Craft code is code whose correctness, structure and cost of change can be understood and defended.**

A change is not good merely because it compiles, tests pass, it is short, it uses fashionable patterns, it was generated quickly, it looks "clean", or an AI agent declares it production-ready.

There is no meaningful guarantee of "perfect code". The engineering target is stronger and checkable:

> Every accepted change must preserve or improve the health of the codebase, and must carry evidence proportional to its risk.

This follows Google's code review standard: do not optimise for theoretical perfection; prevent the codebase from getting worse.

## 1.1 The five properties, in priority order

When they conflict, higher wins. This ordering is the tie-breaker used everywhere in this standard.

| # | Property | Question it answers |
|---|---|---|
| 1 | **Correct** | Does it do what it claims, including on failure paths? |
| 2 | **Safe** | Can it be misused, abused or leak? Are invariants enforced? |
| 3 | **Clear** | Can a competent peer predict its behaviour after one read? |
| 4 | **Simple** | Is this the least conceptual machinery that fully solves the requirement? |
| 5 | **Fast enough** | Does it meet the stated budget without accidental pathologies? |

Maintainability and operability are *consequences* of 1–4, not separate goals to design for directly.

## 1.2 Authoritative conflict-resolution order

When any two rules, conventions or preferences disagree, resolve in this order. This list overrides local instinct, and is the single authoritative version — `02-PROTOCOL.md` and `03-GATE.md` refer to it rather than restating it.

1. **Safety, security, data integrity, legal/licensing constraints.**
2. **Explicit user or product requirement.**
3. **Existing public contract** (API, schema, wire format, CLI, file format) — unless intentionally and explicitly changed.
4. **Existing project convention and architecture** — unless the convention is itself unsafe or is what is being changed.
5. **The principles in this document.**
6. **Heuristics, smells, metrics, scores.**
7. **Aesthetic preference of the author or of the model.** Never sufficient reason on its own.

A lower item must not override a higher one without stated evidence.

---

# 2. Complexity: the operational definition

Adapted from John Ousterhout, *A Philosophy of Software Design*.

> **Complexity is anything about the structure of a system that makes it hard to understand or modify.**

Complexity is not measured by cleverness, line count or number of patterns. It is measured by its **symptoms**:

### Symptom 1 — Change amplification
A simple conceptual change requires edits in many places.
*Test:* "To change the currency rounding rule, how many files must I touch?" More than one usually means the knowledge is duplicated.

### Symptom 2 — Cognitive load
A developer must hold many facts in their head to make a correct change.
*Test:* "How much must I know that is not visible on screen?"

### Symptom 3 — Unknown unknowns *(the worst)*
It is not obvious which code must be changed, or what knowledge is required to change it correctly.
*Test:* "Could a competent developer make this change, believe they were done, and be silently wrong?"

A design that is merely *ugly* but has no unknown unknowns is safer than an elegant design full of them.

## 2.1 The two causes

- **Dependencies** — code that cannot be understood or changed in isolation.
- **Obscurity** — important information that is not obvious from the code.

Every design decision either adds or removes dependencies and obscurity. Judge it on that, not on how it looks.

## 2.2 Complexity is incremental

Complexity is not caused by one catastrophic decision. It accumulates from hundreds of small ones, each individually defensible. This is why "just this once" is the most expensive phrase in engineering.

**Tactical vs strategic:** tactical programming optimises for making this change work now. Strategic programming optimises for the design that will still be workable in fifty changes' time. The correct default is *slightly* strategic: invest a small, bounded amount of extra design effort per change — not a rewrite, not a framework.

---

# 3. Intellectual foundation

This standard deliberately combines engineers with different and sometimes conflicting styles. The goal is not to imitate one programmer, but to keep the principles that survive across styles, languages and decades.

## Linus Torvalds — design around data, and eliminate special cases

> Model the data and its relationships before decorating the solution with code structure.

Torvalds has argued that good programmers care about data structures and their relationships, and described Git as designed around stable, understandable data structures. His notion of **"good taste"** is more specific and more useful than it first sounds: *good taste is removing the special case by choosing a better representation*.

```text
# Bad taste — the special case is handled
remove(list, node):
    if node is list.head:
        list.head = node.next
    else:
        prev = find_previous(list, node)
        prev.next = node.next

# Good taste — the special case cannot occur
remove(list, node):
    indirect = address_of(list.head)
    while deref(indirect) is not node:
        indirect = address_of(deref(indirect).next)
    deref(indirect) = node.next
```

The second version is not clever. It has *fewer states*. That is the point.

Consequences:

- Data models are architecture, not an afterthought.
- Do not compensate for a poor data model with layers of procedural code.
- Make invalid states difficult or impossible to represent when the language allows it.
- Prefer direct relationships over synchronisation between redundant representations.
- **Before adding a branch, ask whether a different representation removes the need for the branch.**
- Schema, types, ownership and lifecycle deserve design attention before helpers and class hierarchies.

Linux kernel style also treats excessive nesting, excessive local state and complex functions as warning signs. These are **signals**, not line-count laws.

## Fred Brooks — essential vs accidental complexity

Some complexity is inherent in the problem (**essential**); some is created by our tools, structures and choices (**accidental**). Craft work is the systematic removal of accidental complexity, and the honest acceptance of essential complexity.

- If a domain has seven valid states, hiding them behind a boolean does not simplify anything — it moves the complexity to the caller and adds obscurity.
- **Conceptual integrity:** a system designed by one coherent mind is better than one designed by a committee of equally good ideas. Consistency beats local optimality.

## David Parnas — information hiding

Modules are decomposition units for **knowledge**, not for processing steps. Split a system so that each module hides one design decision that is likely to change, and exposes an interface that does not reveal it.

*Test:* "If this decision changes, does the change stay inside this module?" If not, the boundary is drawn in the wrong place.

## John Ousterhout — deep modules

> The best modules provide **powerful functionality behind a simple interface**.

```text
Deep    ┌──────────┐   small interface
        │██████████│   large implementation      → high value
        │██████████│
        └──────────┘

Shallow ┌──────────────────────┐  large interface
        │████                  │  trivial implementation → negative value
        └──────────────────────┘
```

The cost of a module is its interface; the benefit is the functionality it hides. A **shallow module** — a class whose interface is as complicated as its implementation, a wrapper that forwards, a "layer" that renames arguments — costs more than it gives.

Consequences:

- **Every new boundary must reduce the total information a caller needs.** If it does not, delete it.
- Pass-through methods and pass-through variables are a design smell, not a design.
- "Classes should be small" is *not* a principle. Many shallow classes are worse than one deep one.
- **Different layer, different abstraction.** If a layer's interface resembles the layer beneath it, the layer is probably unnecessary.
- Prefer general-purpose interfaces with special-purpose usage over the reverse; a somewhat general interface is usually both simpler and more useful than a heap of special-purpose entry points.
- **Pull complexity downwards.** It is better for a module's implementation to be slightly harder than for all its callers to be slightly harder.

## Rob Pike — simplicity, measurement, idioms

*Notes on Programming in C* and *Simplicity is Complicated*:

- Do not guess where bottlenecks are; measure before optimising.
- Prefer simple algorithms and data structures until evidence demands more.
- Data organisation frequently determines the natural algorithm.
- **"A little copying is better than a little dependency."**

Consequences:

- No speculative performance hacks; benchmark performance-sensitive decisions.
- Prefer a boring O(n) solution over a clever one when realistic `n` makes it the better engineering choice.
- Code should be idiomatic for its language, not mechanically translated from another ecosystem.

## John Carmack — make state and failure visible; automate detection

Carmack has emphasised aggressive static analysis, and identifies a recurring bug source: *programmers failing to understand the full set of states in which code may execute*.

Consequences:

- Make state transitions explicit; reduce hidden mutation and invisible dependencies.
- Prefer contracts that tools can verify.
- Treat warnings and static-analysis findings as engineering feedback, not noise to silence.
- Do not assume a codebase is clean because developers feel confident about it.
- Do not split code into tiny functions merely to satisfy a metric if the split makes control flow, state or dependencies harder to see.

## Rich Hickey — simple is not easy

- **easy** = familiar, nearby, quick to reach.
- **simple** = *un-complected*: one role, one concept, not braided together with others.

Ask of every construct: **what is braided together here that could be separate?** Common braids: state + identity, policy + mechanism, configuration + behaviour, I/O + logic, time + value.

Consequences:

- Familiarity does not prove good design; a framework that is quick to type can still produce a highly entangled artifact.
- Complexity hidden behind an abstraction still exists.
- We optimise the **artifact**, not the convenience of producing it.

## Kent Beck — simple design and preparatory refactoring

**Four rules of simple design**, applied strictly in this order:

1. Passes its tests (it works).
2. Reveals intention (names and structure say what it means).
3. Contains no duplicated *knowledge*.
4. Has the fewest elements.

Rule 4 never justifies violating rule 2. "Fewer elements" is the last tie-breaker, not the goal.

> **"Make the change easy, then make the easy change."**

If the requested change is hard, the first step is a *separate*, behaviour-preserving refactoring that makes it easy — then the change itself. Two commits, not one tangle.

## Martin Fowler — smells and continuous refactoring

A code smell is an indicator that deserves investigation, not an automatic verdict.

- Long method, duplication, large class, excessive parameters, shotgun surgery, feature envy → ask questions.
- Do not refactor by smell mechanically.
- Improve design continuously in small behaviour-preserving steps.
- *"Any fool can write code that a computer can understand. Good programmers write code that humans can understand."*

## Sandi Metz — avoid the wrong abstraction

> **Duplication is far cheaper than the wrong abstraction.**

Similar-looking code does not automatically represent the same concept.

- Do not abstract because two blocks look alike.
- Prefer temporary duplication until the semantic boundary is understood.
- If a shared abstraction accumulates flags, modes and caller-specific branches, the abstraction is wrong — **inline it back and re-split**, do not add another parameter.
- Sunk cost is not an argument for keeping an abstraction.
- DRY means do not duplicate **knowledge**, not eliminate every repeated token.

## Titus Winters / Software Engineering at Google — code over time

> **Software engineering is programming integrated over time.**

- **Hyrum's Law:** with enough users, every observable behaviour of your system will be depended upon by somebody — regardless of what you documented. Therefore: minimise what is observable.
- Optimise for the reader, and for the reader who arrives in three years without you.
- "Do not merge a change that definitely makes overall code health worse."
- Beware over-engineering for hypothetical futures.

## Edsger Dijkstra — reasoning and cleverness

- Cleverness creates a proof burden that someone must pay later.
- The more states and interactions a design introduces, the stronger the required evidence.
- Correctness is not established by optimism or by one successful execution.

## Tony Hoare — two ways to build

> There are two ways of constructing a design: make it so simple that there are obviously no deficiencies, or so complicated that there are no obvious deficiencies.

Default to the first. The second is what most "enterprise" architecture actually achieves.

## Gall's Law and Chesterton's Fence — two operating constraints

- **Gall's Law:** a complex system that works has invariably evolved from a simple system that worked. A complex system designed from scratch never works and cannot be patched into working. → Do not start with the framework.
- **Chesterton's Fence:** do not remove a fence until you know why it was put there. → Before deleting a check, branch, sleep, retry or comment that looks pointless, find its origin (`git log`, `git blame`, tests, issue tracker). Unexplained code is a question, not garbage.

---

# 4. The CRAFT properties

A production change should satisfy all relevant properties.

## C — Correct

- Requirements are not invented.
- Inputs, outputs, invariants and failure modes are explicit.
- Tests demonstrate important behaviour.
- Negative paths matter as much as happy paths.
- Concurrency, retries, partial failure and idempotency are considered where relevant.

## R — Readable and Reasonable

A competent developer familiar with the language should understand the change without reverse-engineering the author's thought process. Readable does not mean verbose.

Prefer: domain names, direct control flow, explicit contracts, low surprise, local reasoning.
Avoid: clever one-liners that hide behaviour, generic names, architecture astronautics, unexplained magic values, chains of wrappers with no semantic value.

## A — Appropriate Abstraction

An abstraction is justified when it:

1. represents a stable domain or technical concept,
2. removes duplication of **knowledge**, not of characters,
3. gives callers a **smaller** contract than the thing it hides,
4. reduces rather than relocates complexity,
5. has current, not speculative, value.

**One-sentence test:** if you cannot describe the abstraction's responsibility in one sentence without the word "and", it is not one abstraction.

Red flags: `Manager` / `Helper` / `Utils` / `Processor` / `Handler` / `Factory` / `Engine` / `Service` with vague responsibility; boolean flags switching unrelated behaviours; a generic framework built for one concrete use; one implementation behind an interface with no boundary; wrappers that only forward; an inheritance tree created before variation exists.

## F — Focused

A unit of code should have one coherent reason to exist and one coherent reason to change. This applies to functions, modules, packages, services, components, database boundaries and pull requests.

"Focused" is not "tiny". A 60-line straight-line parser may be more focused than six 10-line functions that bounce state through meaningless layers.

## T — Testable, Tool-verifiable and Traceable

Important behaviour must be capable of verification: compiler/type checker, formatter, linter, static analysis, unit/integration/E2E tests, security scanning, benchmarks where material, observability in production paths.

**If something is hard to test, that is design feedback, not a testing problem.** Usually it means hidden dependencies, hidden state, or a boundary in the wrong place. Fix the design before reaching for a mocking framework.

A claim without evidence remains a claim.

---

# 5. Universal rules

## 5.1 Understand before editing

1. Find the entry point.
2. Trace the relevant data flow.
3. Identify the canonical types/data structures.
4. Find existing implementations of similar behaviour.
5. Read nearby tests.
6. Read project architecture and local conventions.
7. Identify public contracts that must remain compatible.
8. Determine the smallest coherent change.

Do not begin by generating new files.

## 5.2 Data before ceremony

Before creating a new class/module/layer, answer: What data exists? Who owns it? What are its invariants? Which states are valid? What relationships exist? What is mutable? What crosses a boundary? What needs persistence? What can be derived instead of stored?

> If a cleaner data model removes code, prefer the cleaner data model.

## 5.3 Make illegal states unrepresentable, and parse instead of validating

Where the language supports it: constrained types, enums/tagged unions, non-null types, validating constructors, database constraints, schema validation.

**Parse, don't validate.** Do not check a value and pass the *unchecked* type onward — convert it once, at the boundary, into a type that carries the proof.

```text
# Validate (weak): the proof is discarded, so every caller re-checks or forgets to
def send(email: str):
    if not is_valid(email): raise ...
    ...

# Parse (strong): invalid values cannot reach the domain at all
def send(email: EmailAddress):   # EmailAddress can only be constructed from a valid string
    ...
```

Consequences: no re-validation scattered through layers, no "can this be null here?", fewer defensive branches, and the type system enforces the invariant for free.

Do not spread stringly-typed domain state across the codebase.

## 5.4 Prefer local reasoning

A developer should not need to inspect ten unrelated files to understand one simple behaviour.

Prefer: explicit inputs, explicit outputs, narrow dependencies, local invariants, one canonical representation.
Avoid: hidden globals, ambient mutable state, action at a distance, implicit registration, monkey patching, configuration that silently changes semantics.

## 5.5 Keep dependency direction intentional

Core/domain logic should not casually depend on UI framework details, transport formats, database clients, vendor SDKs or process globals. Dependencies point from volatile toward stable, from outer toward inner.

Use boundaries where they buy testability, replaceability or conceptual clarity. Do **not** create boundaries because a pattern catalogue says one should exist.

## 5.6 Design errors out of existence, then make the rest explicit

**Step 1 — eliminate.** The best error is one that cannot occur. Redefine the semantics so the "error" becomes a normal case:

- deleting a non-existent file → success (the postcondition holds either way);
- substring with out-of-range indices → clamp rather than throw;
- an empty collection → a valid input, not an exception;
- an idempotent write → repeating it is not an error.

Every exception you delete this way removes a branch from every caller, forever.

**Step 2 — classify what remains.**

| Kind | Meaning | Handling |
|---|---|---|
| **Expected** | Part of normal operation (not found, invalid input, conflict) | Model it in the return type/signature. It is part of the public contract. |
| **Unexpected** | A broken invariant or bug | Fail fast, loudly, with context. Do not "handle" it. |
| **Transient** | External, may succeed later | Retry only if the operation is idempotent, with bounds and backoff. |

**Step 3 — never launder.** Do not widen a precise failure into `null`, `-1`, `false`, an empty list or a generic "something went wrong". Ambiguous returns push undecidable questions onto the caller.

Prefer: typed/structured errors, meaningful context added at each boundary, correct propagation, observable failures, no sensitive data in logs.

**Aggregate exception handling.** Handle many low-level failures in one place with one policy (a request middleware, a job wrapper), rather than sprinkling `try/catch` at every call site.

## 5.7 Avoid speculative generality

Do not implement imagined providers, unused hooks, "future-proof" plugin systems, flags for hypothetical requirements, configuration nobody requested, or extension points with no current extension.

Solve the known problem. Leave the system easy to change when the next real problem arrives. **"Maybe later" is not a justification.**

## 5.8 Do not worship DRY

> Do these two pieces of code represent the same knowledge, and must they change together for the same reason?

If yes → consider abstraction. If they merely look similar → keep them separate until the shared concept is clear.

**Rule of three:** two occurrences are a coincidence; three are evidence. Wait for the third unless the duplicated thing is a domain rule whose divergence would be a bug (e.g. tax calculation, auth check) — those must be unified immediately.

## 5.9 Do not worship small functions

Split when a semantic boundary becomes clearer. Do not split because a linter says 20 lines or a blog says five.

> A function is too complex when its behaviour, state or invariants cannot be held comfortably in one reading — not when it exceeds a line count.

Corollary: extracting a fragment used **once**, that requires the reader to jump elsewhere and back, usually increases complexity. Extract *deep* pieces (much hidden, little interface), not arbitrary slices.

## 5.10 Names carry domain meaning

- Use the vocabulary of the problem domain: `calculateInvoiceTotal`, not `processData`; `subscriptionRenewalDate`, not `dateValue`.
- **Precision beats brevity.** If a name is hard to choose, the underlying concept is usually unclear — that is design feedback.
- **Length is proportional to scope.** `i` in a three-line loop is fine; `i` as a module-level variable is not.
- **Consistency:** the same concept always gets the same word; different concepts never share a word.
- **Names must not lie.** `getUser` must not create a user. `isValid` must not mutate. A misleading name is worse than a vague one.
- Prefer positive booleans (`isEnabled`, not `isNotDisabled`).
- **Command–Query Separation:** a function either changes state or answers a question, not both, where practical.

## 5.11 Comments explain what code cannot

Write comments for: why a strange constraint exists, why an apparently simpler approach is unsafe, non-obvious domain rules, external-system quirks, invariants, performance evidence, security reasoning, links to the issue/spec/RFC.

Delete comments that narrate syntax.

**Comments as a design tool (Ousterhout):** write the interface comment *before* the implementation. If it is long, full of conditions, or leaks implementation details, the interface is wrong — fix the design, not the comment.

**A stale comment is a bug.** If a change makes a comment untrue, updating it is part of the change.

## 5.12 No warning debt by default

New code must not normalise compiler warnings, linter warnings, failing static analysis, ignored type errors, skipped tests, flaky tests or security findings.

A suppression is acceptable only when all four hold: (1) the checker genuinely cannot model valid behaviour, (2) the scope is as narrow as the language allows, (3) the reason is documented inline, (4) a safer alternative was considered and rejected in writing.

## 5.13 Minimise change surface

One conceptual change, small diff, refactoring separated from behaviour change when practical. Smaller changes are easier to reason about, test, review and revert.

**"Smallest coherent" cuts both ways.** It forbids gold-plating *and* it forbids a hack that leaves an inconsistent state, a duplicated rule, or a lie in a name. Smallest **correct**, not smallest **typed**.

## 5.14 Side effects belong at boundaries — functional core, imperative shell

Where practical: keep domain transformations deterministic and pure; isolate network, filesystem, database, clock and randomness at the edges; inject unstable dependencies explicitly; make transaction boundaries obvious.

This is not a mandate for functional programming. It is a mandate for understandable state — and it is also what makes tests fast and deterministic without mocks.

## 5.15 Performance: design vs tuning

Two different activities, routinely confused. Only the second one needs a benchmark first.

**Design-time (do it now, no permission needed):**
- choose the right data structure and access pattern;
- avoid accidental quadratic behaviour and N+1 queries;
- avoid unbounded growth where input can grow;
- do not re-fetch or re-parse in a loop;
- know the orders of magnitude (memory ≪ SSD ≪ network ≪ cross-region).

Picking a hash map over a linear scan is not premature optimisation; it is choosing a correct data structure.

**Tuning (requires evidence):**
1. define the workload → 2. measure the baseline → 3. identify the bottleneck → 4. change the smallest relevant area → 5. measure again → 6. preserve correctness → 7. record trade-offs.

Caching, pooling, memoisation, concurrency, batching and low-level tricks belong here. Never trade clarity for theoretical performance. **No benchmark, no quantitative claim.**

## 5.16 Security is part of correctness

For externally reachable or sensitive systems, correctness includes authentication, authorisation, input validation at the trust boundary, output encoding, secrets handling, dependency risk, least privilege, safe defaults, rate/abuse considerations, auditability.

Rules that do not bend: authorisation is enforced at the trusted boundary (never only client-side); untrusted input is parsed into a safe type at the edge; never invent cryptography; secrets never enter code, logs, errors or the repository. OWASP ASVS is the default reference.

## 5.17 Idiomatic code beats AI-generic code

A TypeScript solution should look like strong TypeScript; a Go solution like strong Go; a Rust solution like strong Rust; a Python solution like strong Python. Do not translate Java architecture into every language.

Before inventing a pattern: read official language guidance, read mature code already in the repository, use ecosystem conventions, use the standard library.

## 5.18 Deleting code is progress

Removed code has no bugs, no tests to maintain, no security surface and no upgrade cost. A change that deletes more than it adds while satisfying the requirement is usually the better change.

Check every change for code it makes dead — and (Chesterton's Fence) confirm it is truly dead before removing it.

## 5.19 Make interfaces easy to use correctly and hard to use wrongly

- Make the correct call the shortest one; make dangerous operations verbose and explicit.
- Prefer distinct types to positional booleans (`send(msg, urgent=True)` invites `send(msg, True)`).
- Prefer safe defaults; no security-relevant behaviour should depend on the caller remembering a flag.
- Prefer compile-time failure over runtime failure; runtime failure over silent wrong results.
- Minimise what is observable (Hyrum's Law): what you expose, you will maintain forever.

## 5.20 A decision system never lowers the standard

Agents and automated tooling increasingly choose their own next step. That is a question of cost, not of licence: it changes how work is scheduled, never what the work must satisfy.

- **Deterministic rules outrank probabilistic judgement.** Whatever code can decide, code decides. Asking a model a question the runtime has already answered adds cost and a failure mode, and removes nothing.
- **Risk and reversibility control autonomy** — not how certain the actor feels. A one-character change to an authorisation predicate is not made safe by confidence (§8).
- **Confidence is not evidence.** A probability is a statement about a model's preference, not about the world. Only executed checks establish that something works.
- **A decision is not a result.** Choosing to act, and having acted correctly, are different claims with different proof requirements.
- **Speed is bought from deliberation, never from engineering.** Skipping thought on a trivial routing step is efficiency; skipping recon, tests, review or the gate is slop with a faster clock.

`05-DECIDE.md` specifies how to apply this in an agent loop. Nothing in it relaxes this document.

---

# 6. AI slop: operational definition

**AI slop is code produced with insufficient understanding of the repository, domain, constraints or proof of correctness — usually compensated for by excessive surface-level code.**

Slop is defined by properties, not by authorship. A human can write slop. An AI can write craft code.

## Signatures

**Repository amnesia** — duplicates an existing function; creates a second canonical type; ignores an established abstraction; reimplements an available utility.

**Assumption filling** — invents requirements, defaults, API behaviour, environment variables, schema fields or fallback behaviour without evidence.

**Abstraction inflation** — interface + factory + implementation for one use; wrapper around wrapper; `manager`/`service`/`helper` layers with no domain meaning; a generic framework built instead of one concrete feature.

**Brute-force patching** — adds condition after condition until tests pass; treats symptoms instead of finding the violated invariant; edits unrelated code to suppress failures.

**Validation theatre** — tests that only confirm mocks; tests of implementation details; disabling a failing test; changing expected output to match a regression; claiming success without running checks.

**Type-system escape** — `any`, broad casts, unchecked deserialisation, `@ts-ignore`, `# type: ignore`, `unsafe`, nullable-everything — used to silence design problems rather than model them.

**Error laundering** — empty catches; catch-log-continue with no semantics; returning `null` for unrelated failures; converting precise failures into generic messages.

**Defensive noise** — null checks on values that cannot be null; `try/except` around code that cannot fail; re-validating what the type already guarantees; fallbacks that hide the real failure.

**Comment sludge** — paraphrasing each line; long prose explaining obvious code; `# Step 1: … # Step 2: …` narration; fabricated certainty; stale narratives.

**Dependency sprawl** — a package for trivial functionality; a dependency chosen without checking the existing stack; overlapping libraries.

**Diff pollution** — reformatting unrelated files; renaming unrelated symbols; "cleaning up" the area while implementing one feature.

**Configuration inflation** — new env vars, feature flags and options nobody asked for; behaviour that can only be understood by reading config.

**Documentation inflation** — a 300-line README for a 40-line change; emoji headers; marketing tone; "🚀 production-ready", "enterprise-grade", "blazingly fast" — especially without benchmarks.

**Placeholder delivery** — `TODO`, `pass`, stubs, fake data paths or "implementation left as an exercise" presented as complete work.

**Cargo-cult patterns** — repository, mediator, factory, event bus, microservice, CQRS, DTO mapping layer, DI container — added because the pattern sounds professional rather than because the system needs it.

**One item may be legitimate. A cluster is a strong slop signal.**

---

# 7. Slop test

1. Can the author state the domain invariant the change preserves?
2. Can each new abstraction be justified in one sentence without "and"?
3. Can each new dependency be justified?
4. Is there exactly one canonical representation of each important concept?
5. Is failure behaviour explicit?
6. Can the behaviour be tested without knowing implementation details?
7. Did the change avoid unrelated cleanup?
8. Were existing patterns inspected first?
9. Is the code idiomatic for this language?
10. Does the diff get simpler if one layer/helper/interface is deleted?
11. Is any complexity present solely because "we may need it later"?
12. Is performance claimed without measurement?
13. Are static-analysis or type warnings being hidden?
14. Could another developer understand the change from code + tests + short context?
15. Is the codebase at least as healthy as before?
16. **Would a competent peer be able to tell if this silently broke in production?**

If several answers are bad, the change is not craft.

---

# 8. Evidence proportional to risk

Not every change deserves the same ceremony. Ceremony applied to a trivial change is itself a form of waste.

| Risk | Examples | Minimum evidence |
|---|---|---|
| **Low** | copy change, isolated style fix, obvious dead-code removal, comment fix | format, lint/typecheck, targeted test or build |
| **Medium** | business logic, API behaviour, compatible schema change, new endpoint | relevant unit/integration tests, static checks, edge-case review, public-contract check |
| **High** | auth, payments, migrations, concurrency, distributed consistency, cryptography, destructive ops, hot paths, multi-tenant boundaries | design review, stronger coverage, failure-injection, rollback plan, migration validation, security review, benchmarks/load tests, observability, staged rollout |

Risk is determined by **blast radius and reversibility**, not by diff size. A one-character change to an authorisation predicate is a high-risk change.

**One-way doors** (irreversible: data migrations, deletions, published API contracts, wire formats, key rotation, anything users can build on) get design effort *before* implementation. Reversible decisions get decided quickly and revisited if wrong.

---

# 9. What this standard explicitly rejects

Rejected as universal laws:

- "Every function must be under N lines." / "Every class must have one method."
- "Never duplicate code." / "Always use interfaces." / "Always use dependency injection." / "Always use design patterns."
- "More abstraction means more professional code." / "Smaller classes are always better."
- "More tests automatically means better tests." / "100% coverage proves correctness."
- "A type checker proves correctness." / "Passing CI means the design is good."
- "Premature optimisation is the root of all evil" used to justify an O(n²) loop over unbounded input.
- "AI-generated means bad." / "Human-written means good."
- "The model was 96% confident" offered as evidence that something works.

Engineering judgment remains required.

---

# 10. Source lineage

A synthesis, not a claim that every source agrees with every rule.

- Linus Torvalds — Git mailing-list discussion on designing around data structures; the "good taste" talk (TED, 2016).
- Linux Kernel Coding Style — https://docs.kernel.org/process/coding-style.html
- Rob Pike — *Notes on Programming in C*; *Simplicity is Complicated* — https://go.dev/talks/2015/simplicity-is-complicated.slide
- Go project — *Effective Go* — https://go.dev/doc/effective_go
- John Ousterhout — *A Philosophy of Software Design*, 2nd ed.
- David Parnas — *On the Criteria To Be Used in Decomposing Systems into Modules* (1972).
- Fred Brooks — *No Silver Bullet*; *The Mythical Man-Month*.
- John Carmack — *Static Code Analysis*; writings on inlined code and functional style.
- Rich Hickey — *Simple Made Easy* — https://www.infoq.com/presentations/Simple-Made-Easy/
- Kent Beck — *Extreme Programming Explained*; four rules of simple design; "make the change easy, then make the easy change".
- Martin Fowler — *Refactoring*; *Code Smell* — https://martinfowler.com/bliki/CodeSmell.html ; *Agentic Programming* — https://martinfowler.com/bliki/AgenticProgramming.html
- Sandi Metz — *The Wrong Abstraction* — https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction
- Titus Winters, Tom Manshreck, Hyrum Wright — *Software Engineering at Google*; Hyrum's Law — https://www.hyrumslaw.com/
- Michael Feathers — *Working Effectively with Legacy Code* (characterisation tests, seams).
- Daniel Kahneman — *Thinking, Fast and Slow* (the System 1 / System 2 split used in `05-DECIDE.md`).
- Alexis King — *Parse, Don't Validate*.
- Scott Meyers — *Making Interfaces Easy to Use Correctly and Hard to Use Incorrectly*.
- Google Engineering Practices — https://github.com/google/eng-practices
- OWASP ASVS — https://owasp.org/www-project-application-security-verification-standard/
- ISO/IEC 25010 software product quality model.

---

# 11. Final rule

When uncertain, choose the implementation that makes the system **easier to understand truthfully**.

Not the one that looks most architectural.
Not the one with the fewest lines.
Not the one an AI can generate fastest.

The goal is software that competent engineers can reason about, verify, operate and change.
