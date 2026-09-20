---
name: power-of-ten-typescript
description: "Use when writing safety-critical TypeScript. Ten rules."
version: 1.0.0
author: Jorge Quijano (JorgeQuijano), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [TypeScript, Safety-Critical, Static-Analysis, Lint, Code-Review]
---

# Power of 10 for TypeScript

Ten constraints adapted from Gerard Holzmann's JPL "Power of 10" rules for safety-critical code. Each rule states the constraint, the reason, and the automated check that enforces it. The rules narrow what the compiler and linter are allowed to accept — they are not a style guide, and they do not replace type checking.

## When to Use

- Writing or reviewing TypeScript where a defect is expensive: payments, auth, data pipelines, device control, anything long-running or unattended.
- Asked to harden, audit, or make a module reliable.
- **Don't use for:** prototypes, spikes, throwaway scripts, UI glue, or code about to be deleted. These rules cost lines and add friction; applying them to a spike is waste.
- **Don't use as a rewrite mandate.** Apply to new or touched code and say so; retrofitting a whole codebase at once produces thousands of findings and no safer code (see Pitfalls).

## Prerequisites

- Node ≥ 20.9, TypeScript ≥ 5.0, ESLint ≥ 9 (flat config), typescript-eslint ≥ 8.
- Verified on Node 22.22, tsc 6.0.3, ESLint 10.11, typescript-eslint 8.70.
- Copy-ready configs: `references/checks.md`. Corrected before/after code: `references/examples.md`.

## How to Run

Both gates, every time, before declaring TypeScript work done:

```
npx tsc --noEmit                 # Rule 10
npx eslint . --max-warnings 0    # Rules 1, 2, 4, 5, 6, 7, 8, 9, 10
```

A non-zero exit from either gate means the work is not done. Never `eslint-disable` a Power-of-10 rule to get green — fix the code, or delete the finding with a written reason.

## The Ten Rules

Each rule: constraint, why, and the check that enforces it. Examples in `references/examples.md`.

**1. Simple control flow.** No recursion, direct or indirect — rewrite as a loop with an explicit work queue. No callbacks nested more than two deep; use `async`/`await`. Guard clauses over nested `if`s; max block depth 3. *Why:* recursion depth and callback tangles are where stack overflows and unreachable error paths hide. *Check:* `max-depth`, `max-nested-callbacks`; recursion is a review check — search the diff for a function that calls itself.

**2. Bounded loops.** Every loop has a bound you can state in one sentence. No `while (true)`, no `for (;;)`, no exit that depends on an external party. Bound the data too: indexed access yields `T | undefined`. *Why:* an unbounded loop is a hang, and a hang in production is an outage. *Check:* `no-constant-condition` (`checkLoops: 'all'`); tsconfig `noUncheckedIndexedAccess`.

**3. Bounded memory.** No unbounded accumulation: arrays or maps that grow with input volume, caches without an eviction limit, "collect everything then process" over a stream. Page or stream instead; in a *measured* hot path, reuse a fixed buffer. *Why:* this is the one JPL rule that cannot survive translation literally — JavaScript allocates constantly and the GC is optimised for short-lived objects, so the property worth enforcing is that growth is bounded, not that allocation is absent. *Check:* review — `.push(` or `new Map()` inside a loop over unbounded input, or a cache with no max size. Apply the buffer half only where a profile shows GC pressure, and name the profile.

**4. Small functions.** ≤ 40 lines (JPL allows 60; 40 is the practical TypeScript ceiling), cyclomatic complexity ≤ 10, ≤ 4 parameters (options object beyond that), ≤ 300 lines per file. *Why:* reviewability — a function that fits on one screen can be reasoned about completely. *Check:* `max-lines-per-function`, `complexity`, `max-params`, `max-lines`.

**5. Assertions, not assumptions.** Every value crossing a trust boundary (network, storage, config, user input, untyped dependency) is parsed by a schema — `zod`, `valibot`, or a hand-written narrowing function — and only the parsed result is used. `as`, `!`, and `@ts-ignore` are not validation: zero of them. *Why:* unvalidated input is the root of most incident classes, and a cast is the same assumption with worse error messages. *Check:* `consistent-type-assertions: never`, `no-non-null-assertion`, `no-explicit-any`, `ban-ts-comment`; "every boundary has a parse" is a review check.

**6. Smallest possible scope.** `const` by default, no `var`, declare at the point of use, no module-level mutable state. No shadowing, no unused variables or parameters (prefix deliberately-unused ones with `_`). *Why:* a value that cannot be reached cannot be corrupted, and unreachable state is state you cannot reason about. *Check:* `prefer-const`, `no-var`, `no-shadow`, `prefer-readonly`, `no-unused-vars`.

**7. Check every return value and parameter.** Await every promise, or discard it explicitly with `void` plus a comment saying why. Consume, log, or assert every non-void return. Handle every union member — exhaustive `switch`, no fallthrough. Compare explicitly (`value !== ''`, `list.length > 0`) instead of leaning on truthiness. *Why:* the ignored return value is the classic silent failure; this is the highest-yield rule in the set. *Check:* `no-floating-promises`, `no-misused-promises`, `require-await`, `switch-exhaustiveness-check`, `strict-boolean-expressions`; tsconfig `noFallthroughCasesInSwitch`.

**8. No metaprogramming.** No `eval`, `new Function`, string-based timers, dynamic `require`, or string-keyed prototype manipulation. Static imports only. Configuration is data validated by a schema, not code that builds code. *Why:* a string the analyzer cannot resolve is a code path no tool can check. *Check:* `no-eval`, `no-new-func`, `no-implied-eval`, `no-require-imports`.

**9. No mutation, shallow indirection.** Return new values instead of mutating parameters or shared objects (`{ ...user, name }`); mark fields and arrays `readonly`; accept `readonly T[]`. Keep property chains to ≤ 2 levels and at most one `?.` — flatten the type instead of chasing `a.b.c.d`, and treat a missing value as a branch you handle explicitly. *Why:* mutation is invisible at the call site, and a long chain is an unhandled absence waiting to throw. *Check:* `no-param-reassign` (`props: true`), `prefer-readonly`; mutation through method calls (`arr.push`, `map.set`) and chain depth are review checks.

**10. Strict compilation, zero warnings.** `strict` plus the hardening flags, `--max-warnings 0`, and no suppressions: `@ts-ignore` and `@ts-nocheck` are banned, `@ts-expect-error` only with a written reason on the same line. Widen deliberately (`| undefined`), never to `any`. *Why:* a warning is a defect you have agreed to keep. *Check:* both gates exit 0.

## Pitfalls

- **Rule 3 is a translation, not a transcription.** JPL bans allocation after init; JavaScript cannot honour that. Do not micro-optimise allocation — fix unbounded growth. Citing Rule 3 to justify pooling in a path nobody profiled adds bugs, not safety.
- **Two rules are review-only.** Recursion (Rule 1) and mutation through method calls (Rule 9) have no reliable lint rule. The config does not prove them; the diff does. Do not claim the gates cover all ten.
- **`as` is banned, `as const` is not.** Verified: `assertionStyle: 'never'` permits const assertions, so literal-union constants keep working. Reach for `satisfies` where you would have cast to widen a type.
- **Do not turn all ten on at once on an existing codebase.** Order: (1) Rule 10 + `no-floating-promises` — the real bugs; (2) Rules 1, 4, 6 — mechanical, mostly autofixable; (3) Rules 5 and 9 — real edits at boundaries. Land each step separately.
- **`--max-warnings 0` turns legacy warnings into a blocked pipeline.** Fix or delete the warning; suppressing it to unblock CI is the failure this rule exists to prevent.
- **`index.html` in this repo restates the rule titles for the web page.** If a title changes here, change it there too, or the published page drifts from the skill.

## Verification

- [ ] `npx tsc --noEmit` exits 0
- [ ] `npx eslint . --max-warnings 0` exits 0 with zero warnings
- [ ] No `as T`, `!`, `@ts-ignore`, or `eslint-disable` added in the diff
- [ ] Every trust boundary parses its input with a schema before use
- [ ] Every loop's bound is statable in one sentence; no unbounded accumulation added
- [ ] No function over 40 lines, complexity over 10, or file over 300 lines added
- [ ] No new module-level mutable state
- [ ] Every promise awaited or `void`ed with a reason; every union handled exhaustively
- [ ] No recursion added (grep the diff for self-calls)
