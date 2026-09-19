# Power of 10: configs, rule to check matrix, divergences from JPL

Verified against Node 22.22, TypeScript 6.0.3, ESLint 10.11, typescript-eslint 8.70.
Smoke-test evidence is in section 6; it was produced by running the configs below, not by reading them.

## 1. tsconfig.json (Rule 10)

```json
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "useUnknownInCatchVariables": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,
    "skipLibCheck": true
  }
}
```

Keep your own `target`, `module`, `moduleResolution`, and `lib` — those follow the runtime, not this rule set.
`skipLibCheck: true` is deliberate: third-party `.d.ts` errors are not your code and you cannot fix them. Vendor your types first if you want it off.
`verbatimModuleSyntax` plus `isolatedModules` keeps type-only imports explicit, which is what makes single-file transpilers safe.
`exactOptionalPropertyTypes` and `noUncheckedIndexedAccess` are the two flags that actually change behaviour (they catch passing `undefined` explicitly and unguarded index access) — the rest are hygiene.

## 2. eslint.config.js (flat config, ESLint 9+)

```js
import tseslint from 'typescript-eslint';

export default tseslint.config(
  { ignores: ['node_modules/**', 'dist/**'] },
  {
    files: ['**/*.ts'],
    extends: [...tseslint.configs.strictTypeChecked],
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    rules: {
      // Rule 1 - simple control flow
      'max-depth': ['error', 3],
      'max-nested-callbacks': ['error', 2],
      // Rule 2 - bounded loops
      'no-constant-condition': ['error', { checkLoops: 'all' }],
      // Rule 4 - small functions
      'max-lines': ['error', { max: 300, skipBlankLines: true, skipComments: true }],
      'max-lines-per-function': ['error', { max: 40, skipBlankLines: true, skipComments: true }],
      complexity: ['error', 10],
      'max-params': ['error', 4],
      // Rule 5 - assert, do not cast
      '@typescript-eslint/consistent-type-assertions': ['error', { assertionStyle: 'never' }],
      '@typescript-eslint/no-non-null-assertion': 'error',
      '@typescript-eslint/no-explicit-any': 'error',
      // Rule 6 - smallest possible scope
      'prefer-const': 'error',
      'no-var': 'error',
      '@typescript-eslint/no-shadow': 'error',
      // Rule 7 - check every return value and parameter
      '@typescript-eslint/no-floating-promises': 'error',
      '@typescript-eslint/no-misused-promises': 'error',
      '@typescript-eslint/require-await': 'error',
      '@typescript-eslint/switch-exhaustiveness-check': 'error',
      '@typescript-eslint/strict-boolean-expressions': 'error',
      // Rule 8 - no metaprogramming
      'no-eval': 'error',
      'no-new-func': 'error',
      '@typescript-eslint/no-implied-eval': 'error',
      '@typescript-eslint/no-require-imports': 'error',
      // Rule 9 - no mutation, shallow indirection
      'no-param-reassign': ['error', { props: true }],
      '@typescript-eslint/prefer-readonly': 'error',
      // Rule 10 - zero warnings, no suppressions
      '@typescript-eslint/ban-ts-comment': [
        'error',
        { 'ts-ignore': true, 'ts-nocheck': true, 'ts-expect-error': 'allow-with-description' },
      ],
    },
  },
);
```

`strictTypeChecked` carries the `no-unsafe-*` family, `no-unnecessary-condition`, and `restrict-template-expressions`; the explicit list above is only what it does not already enable.
`projectService: true` is what makes the type-aware rules work without listing every file in `parserOptions.project`.
Note `no-param-reassign` needs `props: true` — the default only catches `param = x`, not `param.field = x`.

## 3. Rule to check matrix

- Rule 1 Simple control flow — `max-depth`, `max-nested-callbacks` — lint (recursion: review)
- Rule 2 Bounded loops — `no-constant-condition` (`checkLoops: 'all'`), `noUncheckedIndexedAccess` — lint + tsc
- Rule 3 Bounded memory — review only (see Pitfalls in SKILL.md) — no automatic check
- Rule 4 Small functions — `max-lines-per-function`, `complexity`, `max-params`, `max-lines` — lint
- Rule 5 Assertions not assumptions — `consistent-type-assertions: never`, `no-non-null-assertion`, `no-explicit-any`, `ban-ts-comment` — lint (schema coverage: review)
- Rule 6 Smallest scope — `prefer-const`, `no-var`, `no-shadow`, `prefer-readonly`, `no-unused-vars` — lint
- Rule 7 Check every value — `no-floating-promises`, `no-misused-promises`, `require-await`, `switch-exhaustiveness-check`, `strict-boolean-expressions`, `noFallthroughCasesInSwitch` — lint + tsc
- Rule 8 No metaprogramming — `no-eval`, `no-new-func`, `no-implied-eval`, `no-require-imports` — lint
- Rule 9 No mutation, shallow indirection — `no-param-reassign` (`props: true`), `prefer-readonly` — lint (`arr.push` / `map.set` on inputs, chain depth: review)
- Rule 10 Strict compilation — `strict` + hardening flags, `--max-warnings 0` — tsc + lint

## 4. What the config cannot see

Recursion, mutation through method calls (`array.push`, `map.set`, `set.add` on a parameter or shared object), property-chain depth, whether a trust boundary actually parses its input, and unbounded accumulation. These are diff-review checks — state them as such instead of claiming the gates cover all ten rules.

`no-restricted-syntax` selectors can nominally catch `arr.push(...)`, but they cannot tell a local accumulator from a shared object, so they fire on correct code. Do not add them.

## 5. Divergences from JPL's Power of 10

- JPL 1 (no recursion, no goto) — kept, plus a callback-depth limit, since nested callbacks are the JS form of the same tangle.
- JPL 2 (fixed loop bounds) — kept.
- JPL 3 (no dynamic allocation after init) — **changed** to bounded memory/growth. JavaScript allocates on almost every expression and the GC is optimised for short-lived objects; enforcing the letter of the rule is impossible and the spirit ("memory use stays predictable") is what bounds. This is the one rule where the TypeScript version is a different rule, and it is stated as review-only for that reason.
- JPL 4 (one page, 60 lines) — kept, tightened to 40 lines plus complexity and parameter limits. TypeScript functions are denser than C, and 40 lines is roughly what fits on a phone-sized review screen.
- JPL 5 (two assertions per function, side-effect free) — **retargeted** to schema parsing at trust boundaries. Asserting invariants inside every small function in TypeScript mostly reproduces what the type checker already proves, whereas unvalidated external input is a real incident class. Where a function does assert, the assertion stays side-effect free.
- JPL 6 (smallest scope) — kept verbatim in intent.
- JPL 7 (check all return values and parameters, validate them) — kept; it is the highest-yield rule and the one `no-floating-promises` enforces best. `strict-boolean-expressions` is the parameter side.
- JPL 8 (limited preprocessor) — retargeted to metaprogramming: `eval`, `new Function`, dynamic `require`. TypeScript has no preprocessor; these are the constructs the rule was protecting against.
- JPL 9 (one level of dereference, no function pointers) — retargeted to immutable updates and shallow property chains, and the original title's "restrict mutation" claim is now actually the rule's content.
- JPL 10 (all warnings, two static analyzers, zero warnings, no suppressions) — kept; the two analyzers become `tsc` plus the type-aware ESLint pass.

## 6. Smoke-test evidence

Run in a scratch project on the configs above, with one deliberately-violating file and one clean file:

- `npx tsc --noEmit` → exit 0. The violating file is *type-clean*, which is the point: lint is the gate that catches it.
- `npx eslint src/good.ts` → exit 0, 0 findings.
- `npx eslint . --max-warnings 0` → exit 1, 37 errors, all in the violating file.
- 25 distinct rules fired: `ban-ts-comment`, `consistent-type-assertions`, `no-explicit-any`, `no-floating-promises`, `no-implied-eval`, `no-meaningless-void-operator`, `no-non-null-assertion`, `no-shadow`, `no-unnecessary-condition`, `no-unsafe-assignment`, `no-unsafe-call`, `no-unsafe-member-access`, `no-unsafe-return`, `require-await`, `restrict-template-expressions`, `strict-boolean-expressions`, `max-depth`, `max-lines-per-function`, `max-nested-callbacks`, `no-constant-condition`, `no-eval`, `no-new-func`, `no-param-reassign`, `no-var`, `prefer-const`.
- Present in the config but not triggered by the sample (they need cases that also break the build in a sandbox, e.g. a CJS `require`): `no-misused-promises`, `no-require-imports`, `switch-exhaustiveness-check`, `max-params`, `complexity`, `max-lines`, `prefer-readonly`, `no-unused-vars`.

Re-run this smoke test after any config edit; a config that no longer loads reports nothing, which reads exactly like a clean codebase.

## 7. Rollout order for an existing codebase

1. **Rule 10 + `no-floating-promises`.** Fixes real bugs, finds them fast.
2. **Rules 1, 4, 6.** Mechanical; mostly autofixable or extract-function work.
3. **Rules 5, 9.** Real edits at every trust boundary and every mutation site.
4. **Rules 2, 7, 8.** Turn on last where the codebase predates them.

Land each step as its own commit or PR with the gates already green, or the diff becomes unreviewable and the rules get blamed for it.
