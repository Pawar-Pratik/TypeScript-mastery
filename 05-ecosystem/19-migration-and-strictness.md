# Lesson 19 — Migration & strictness strategy

> **Why this lesson exists:** *"how would you migrate a large JavaScript codebase to TypeScript?"* is a standard senior interview question, and it's really testing whether you can execute a long-running technical change **without stopping feature delivery**. The wrong answer is a big-bang rewrite. The right answer is a ratchet — and the same technique applies to enabling `strict` on a codebase that's already TypeScript-but-loose, which is the situation you're far more likely to inherit.

**Time:** ~55 minutes · **Prereq:** Modules 1–4

---

## 1. The idea in one sentence

> **Never migrate by rewriting — install a ratchet: make the codebase measurably stricter in one dimension, lock that gain in CI so it cannot regress, and repeat while feature work continues around you.**

---

## 2. The two migrations

| Situation | Harder part |
|---|---|
| **JS → TS** | Sheer volume; and the temptation to "fix" logic while converting |
| **TS-loose → TS-strict** | `strictNullChecks` on a mature codebase can produce thousands of errors at once |

You'll meet the second far more often. Both use the same technique.

### The rule that makes either survivable
**Convert, don't refactor.** When you change `user.js` to `user.ts`, the *only* goal is that it compiles with the same behaviour. Fixing a bug you spot mid-conversion means your PR mixes a mechanical change with a behavioural one — reviewers can't check it, and if something breaks nobody knows which half caused it.

Write the bug down. Fix it in a separate PR. **Being able to say this out loud is most of the answer to the interview question.**

---

## 3. JavaScript → TypeScript

### Phase 0 — Set up the ratchet (before converting anything)
```jsonc
// tsconfig.json — start permissive so nothing breaks
{
  "compilerOptions": {
    "allowJs": true,            // .js files participate
    "checkJs": false,           // but aren't checked YET
    "strict": false,
    "noEmit": true,
    "target": "ES2022", "module": "ESNext", "moduleResolution": "Bundler",
    "esModuleInterop": true, "skipLibCheck": true, "resolveJsonModule": true
  },
  "include": ["src/**/*"]
}
```
```jsonc
// package.json
{ "scripts": { "typecheck": "tsc --noEmit" } }
```
Add `npm run typecheck` to CI **now**, while it passes trivially. That's the ratchet — every subsequent gain gets locked in automatically.

### Phase 1 — Type the dependencies
```bash
npm i -D typescript @types/node @types/react @types/express …
```
For untyped packages, write minimal `.d.ts` files ([Lesson 14](../04-real-code/14-modules-and-declarations.md)) — type the 5% you use, not the whole API. **Do this before converting your own files**, or every conversion drowns in "cannot find module".

### Phase 2 — Convert leaves first
Build the dependency graph and start at the **leaves** — files that import nothing of yours:
```bash
npx madge --circular --extensions ts,tsx,js,jsx src/     # also finds cycles worth fixing
```
Why leaves first: a converted file's types flow *upward* to its importers, so each conversion makes the next one easier. Converting an entry point first means everything it imports is still `any`, so you type against nothing.

```
utils/ → domain/ → services/ → routes/ → app.ts
  ↑ start here                              ↑ end here
```

### Phase 3 — Convert a file
```ts
// 1. Rename .js → .ts (or .tsx if it has JSX)
// 2. Fix errors with ANNOTATIONS, not logic changes
// 3. Where you genuinely don't know the type yet, be honest and greppable:

type TODO = any;                                   // in a shared types file
function processPayment(input: TODO): TODO { }     // ← grep "TODO" later, not "any"
```
`type TODO = any` is a small trick with real value: it's greppable, it distinguishes "not yet typed" from "deliberately `any`", and it lets a lint rule ban raw `any` immediately.

**Automation that genuinely helps:** `ts-migrate` (Airbnb) converts files and inserts `@ts-expect-error` at each failure, giving you a compiling codebase with a visible debt list. Good for the mechanical bulk; you still review every file.

### Phase 4 — Turn on `checkJs` for the remaining JS
```jsonc
{ "allowJs": true, "checkJs": true }
```
Now un-converted `.js` files are checked too, using JSDoc types:
```js
/** @param {string} id  @returns {Promise<import("./types").Payment>} */
export async function getPayment(id) { }
```
This gets you real type safety on files you haven't converted — and for small libraries or Node scripts, JSDoc + `checkJs` is a perfectly legitimate **end state**, not just a waypoint. (Svelte notably moved *back* to this approach to avoid a build step.)

---

## 4. Loose → strict: the flag order

Enable one flag at a time, in **this order** — it's roughly easiest-to-hardest, and each one reduces the noise in the next.

```
1. noImplicitAny          ← biggest bang, moderate effort
2. strictBindCallApply    ← usually near-zero errors
3. strictFunctionTypes    ← usually few errors
4. noImplicitThis         ← few, mostly in older class code
5. strictPropertyInitialization  ← classes only
6. useUnknownInCatchVariables    ← mechanical: narrow every catch
7. alwaysStrict           ← trivial
8. strictNullChecks       ← ★ THE BIG ONE. Often thousands of errors. Do it last
9. noUncheckedIndexedAccess      ← after strict is stable
10. exactOptionalPropertyTypes   ← last; subtle and noisy
```

**Why `strictNullChecks` is last:** it's typically 70–90% of the total errors, and every other flag fixed first reduces the noise you're wading through. It's also the one that genuinely finds bugs — every error is a place where `null` could arrive and nobody checked.

### Per-directory ratcheting
```jsonc
// tsconfig.strict.json — a second config covering only the strict-clean directories
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "strict": true, "noUncheckedIndexedAccess": true },
  "include": ["src/domain/**/*", "src/schemas/**/*", "src/lib/money.ts"]
}
```
```jsonc
{ "scripts": {
    "typecheck": "tsc --noEmit",
    "typecheck:strict": "tsc --noEmit -p tsconfig.strict.json"
}}
```
Both run in CI. **Every PR that makes a new directory strict-clean adds it to `include`** — and it can never regress. This is the ratchet made concrete, and it's the answer that sounds like you've actually done it.

### The error-count ratchet
```bash
# Record today's count, then fail CI if it ever goes up.
npx tsc --noEmit -p tsconfig.strict-candidate.json 2>&1 | grep -c "error TS" > .type-errors-baseline
```
```yaml
- name: Type errors must not increase
  run: |
    CURRENT=$(npx tsc --noEmit -p tsconfig.strict-candidate.json 2>&1 | grep -c "error TS" || true)
    BASELINE=$(cat .type-errors-baseline)
    echo "current=$CURRENT baseline=$BASELINE"
    [ "$CURRENT" -le "$BASELINE" ] || { echo "❌ Type errors increased"; exit 1; }
```
Crude and extremely effective: the number only goes down. Teams fix errors opportunistically while shipping features, and the count trends to zero without anyone running a migration project.

---

## 5. Handling what fights you

### `@ts-expect-error` over `@ts-ignore` — always
```ts
// @ts-expect-error — legacy API returns unknown shape; typed in TICKET-482
const result = legacyApi.call(input);
```
| | `@ts-ignore` | `@ts-expect-error` |
|---|---|---|
| Suppresses the error | Yes | Yes |
| If the error **goes away** | Silently stays forever | **Becomes an error itself** |

`@ts-expect-error` is self-cleaning: fix the underlying problem and the compiler tells you to delete the suppression. `@ts-ignore` accumulates silently for years. **Ban `@ts-ignore` in ESLint and require a description on every `@ts-expect-error`:**
```jsonc
"@typescript-eslint/ban-ts-comment": ["error", {
  "ts-ignore": true,
  "ts-expect-error": "allow-with-description",
  "minimumDescriptionLength": 10
}]
```

### The three degrees of "I don't know this type"
```ts
const x: any = legacy();          // ❌ infects everything downstream
const y: unknown = legacy();      // ✅ safe, forces a decision at the use site
const z: TODO = legacy();         // ✅ greppable, and bannable later
```
**Prefer `unknown`.** It's the only one where the compiler keeps helping you.

### Libraries that fight you
| Problem | Fix |
|---|---|
| No types at all | `.d.ts` for the 5% you use ([Lesson 14](../04-real-code/14-modules-and-declarations.md)) |
| Wrong/outdated `@types` | `declare module` augmentation, or `patch-package`, or a PR to DefinitelyTyped |
| Types too loose (returns `any`) | Wrap it in your own typed adapter function — one place, validated |
| Two versions of `@types/x` | `npm ls @types/x`, then an `overrides`/`resolutions` entry |

```ts
// The adapter pattern — contain the untyped library at one boundary
// src/lib/legacy-adapter.ts  — the ONLY file allowed to touch the untyped API
import legacy from "untyped-lib";

export async function fetchPayment(id: PaymentId): Promise<Result<Payment, ApiError>> {
  const raw: unknown = await (legacy as any).get(`/payments/${id}`);
  const parsed = PaymentSchema.safeParse(raw);      // ← validate at the boundary (Lesson 15)
  return parsed.success ? Ok(parsed.data) : Err(new ApiError(502, "invalid_response", "…"));
}
```
**This is the pattern to reach for whenever a dependency's types are bad:** one adapter module, one `any`, one validation, and the rest of the codebase sees a properly typed function. It's the boundary discipline applied to libraries instead of the network.

### `ts-reset` — fixing the standard library's honest lies
```ts
// reset.d.ts
import "@total-typescript/ts-reset";
```
Changes several genuinely wrong built-in types:
| Before | After |
|---|---|
| `JSON.parse(s)` → `any` | `unknown` |
| `res.json()` → `any` | `unknown` |
| `arr.filter(Boolean)` → keeps `null` in the type | correctly removes it |
| `arr.includes(x)` → rejects a wider `x` | accepts it |
| `Object.keys(o)` → `string[]` | (deliberately unchanged — see below) |

Worth adopting on any project doing serious boundary work, because the `JSON.parse`/`res.json()` change alone forces the [Lesson 15](../04-real-code/15-the-boundary-and-parsing.md) discipline everywhere.

> **Why `Object.keys` returns `string[]` and not `(keyof T)[]`:** because an object may have *more* properties than its type declares (structural typing — [Lesson 06](../02-type-system/06-structural-typing-and-variance.md)), so `(keyof T)[]` would be unsound. It's a correct lie, not a bug — and it's a great small interview question.

---

## 6. Making it stick: the social half

The technical plan is the easy part. These are what actually determine success:

| Practice | Why |
|---|---|
| **Ratchet in CI from day one** | Without enforcement, gains erode within a sprint |
| **Convert files you're already touching** | Migration rides along with feature work instead of competing with it |
| **No mixed PRs** (conversion + behaviour) | Reviewable, revertable, and blame-able |
| **Publish the number weekly** | "347 files left" / "1,204 → 810 errors" keeps momentum visible |
| **Type the shared/domain layer first** | It benefits every consumer immediately, so people feel the win early |
| **Write the team's rules down** | When `any` is acceptable, when `as` is, what needs validation |
| **Expect a bug-finding spike** | `strictNullChecks` *finds real bugs*. Budget time to fix them, and celebrate them — they're the payoff |

That last row is the one to volunteer in an interview: **a strictness migration surfaces latent production bugs**, and a team that treats those as "migration blockers" rather than "bugs we just found for free" will lose faith in the project.

---

## 7. What "done" looks like

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "allowJs": false            // ← no .js files remain
  }
}
```
```
[ ] Zero `@ts-ignore` (ESLint-enforced)
[ ] Every `@ts-expect-error` has a description and a ticket
[ ] `no-explicit-any` is an error, with a documented exception list
[ ] All external boundaries validated with schemas
[ ] `tsc --noEmit` is a required CI check
[ ] Type-aware ESLint rules enabled
[ ] Team conventions documented in the repo
```

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Ratchet in CI before converting anything** | Enforcement is what makes gains permanent |
| **Convert, don't refactor. Never mix in one PR** | Reviewable and revertable |
| **Leaves first, following the import graph** | Types flow upward; each conversion eases the next |
| **Type dependencies before your own files** | Otherwise every conversion drowns in module errors |
| **One flag at a time; `strictNullChecks` last** | It's 70–90% of the errors |
| **Per-directory strict config that only grows** | The most practical ratchet |
| **`@ts-expect-error` with a description; ban `@ts-ignore`** | Self-cleaning vs silently permanent |
| **`unknown` over `any`; `TODO` alias if you must** | `unknown` keeps the compiler helping |
| **Adapter modules to contain bad third-party types** | One `any`, one validation, typed everywhere else |
| **Adopt `ts-reset`** | Fixes `JSON.parse`/`res.json()` to `unknown` — forces boundary discipline |
| **Budget time for the bugs strict mode finds** | They're the payoff, not an obstacle |
| **Publish progress weekly** | Long migrations die from invisibility |

---

## 9. Interview traps

**Q1. "How would you migrate a 200k-line JavaScript codebase to TypeScript?"**
Structure: (1) tooling + a CI ratchet that passes trivially today; (2) type the dependencies; (3) convert leaves first, following the import graph; (4) `checkJs` + JSDoc for the remainder; (5) enable strict flags one at a time, `strictNullChecks` last; (6) enforce with a per-directory strict config or an error-count ratchet. Then the two rules that make it work: **convert-don't-refactor**, and **ride along with feature work** rather than running a separate migration project.

**Q2. "What order do you enable strict flags?"**
`noImplicitAny` first (biggest value, moderate effort), then the cheap ones, **`strictNullChecks` last** because it's usually 70–90% of the errors and every earlier fix reduces the noise. Then `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` after strict is stable.

**Q3. "`@ts-ignore` or `@ts-expect-error`?"**
`@ts-expect-error`, always — it errors if the underlying problem gets fixed, so suppressions clean themselves up. `@ts-ignore` accumulates silently. Ban it in ESLint and require a description.

**Q4. "A dependency has wrong types. What do you do?"**
In order: augment with `declare module`; write your own `.d.ts` for the part you use; wrap it in a typed **adapter module** that validates at the boundary; `patch-package` as a stopgap; and a DefinitelyTyped PR as the real fix. The adapter is the one that scales — one `any` contained in one file.

**Q5. "How do you stop a migration regressing?"**
CI enforcement: a strict config whose `include` only grows, or an error-count baseline that can only decrease. Without a mechanical gate, gains erode within a sprint because nobody notices a new `any`.

**Q6. "Is JSDoc-typed JavaScript a legitimate destination?"**
Yes — `// @ts-check` + JSDoc uses the *same checker*, with no build step. Genuinely right for small libraries and Node scripts (Svelte moved to it deliberately). TypeScript syntax wins for expressiveness and ergonomics at scale, and Node's native type-stripping narrows the build-step gap further.

**Q7. "Why doesn't `Object.keys(obj)` return `(keyof T)[]`?"**
Because structural typing means an object can have more properties than its type declares, so `(keyof T)[]` would be unsound. It's a deliberate, correct limitation. If you know the object is exact, cast at that one site — or use a helper that documents the assumption.

**Q8. "What's the biggest risk in a strictness migration?"**
Losing team faith. The two ways that happens: a big-bang PR nobody can review, and treating the real bugs `strictNullChecks` uncovers as migration blockers instead of as the payoff. Mitigate with small PRs, a visible number, and explicit time budgeted for the bugs you find.

**Q9. "What's `ts-reset`?"**
A set of declaration overrides fixing genuinely wrong standard-library types — most importantly `JSON.parse` and `res.json()` returning `unknown` instead of `any`, and `filter(Boolean)` narrowing correctly. Adopting it forces the boundary discipline everywhere, which is exactly what you want.

---

## 10. Build & break

### Build — run a real migration
Take a small JavaScript project (yours, or a small open-source one) and migrate it properly:
1. Set up the ratchet and get `typecheck` green with `strict: false`.
2. `npx madge --extensions js,jsx src/` to find the leaves.
3. Convert 5 files, leaves first, in separate commits. **Note every bug you spot and don't fix it** — write them in `FOUND-BUGS.md`.
4. Enable `noImplicitAny`, fix, commit.
5. Enable `strictNullChecks` and **count the errors**. That number is the lesson.
6. Fix 20 of them. How many were real latent bugs?

That last count is the most valuable data point you'll get — and it's the anecdote that makes your interview answer credible.

### Build — the strict ratchet for Ledger
Even in a greenfield project, set up `tsconfig.strict.json` with `exactOptionalPropertyTypes` and `noUncheckedIndexedAccess`, covering only `src/domain` and `src/schemas`. Add both checks to CI. Then grow the `include` list as you go — practising the technique where it's cheap.

### Break — three experiments
1. **`@ts-ignore` vs `@ts-expect-error`.** Suppress an error with each, then fix the underlying problem. One tells you to clean up; one doesn't.
2. **`any` infection.** In a converted file, type one function's parameter `any` and trace how far downstream the type information dies. Then `unknown` and count the errors it surfaces.
3. **`ts-reset`.** Add it to a project using `res.json()` and `JSON.parse`. Count the new errors — every one is an unvalidated boundary.

### Explain out loud (90 seconds)
1. The six phases of a JS→TS migration.
2. The flag order, and why `strictNullChecks` is last.
3. Two ratchet mechanisms.
4. Convert-don't-refactor, and why it matters for reviews.
5. The biggest non-technical risk.

---

## Module 5 complete — checkpoint

- [ ] How to type React props, and the `React.FC` history
- [ ] The context + throwing-hook pattern
- [ ] `as const` on hook tuple returns
- [ ] The generic table-column type
- [ ] Why the dev server doesn't catch type errors
- [ ] The diagnostic sequence for a slow build
- [ ] Three compile-speed fixes and why each works
- [ ] What project references buy and cost
- [ ] The six migration phases and the flag order
- [ ] Two ratchet mechanisms
- [ ] `@ts-expect-error` vs `@ts-ignore`
- [ ] The adapter pattern for bad third-party types

---

## What's next

Module 6 converts everything into interview performance: a rapid-fire question bank, live type challenges (a real interview format you should practise), and a review guide.

Next → **[Lesson 20: Rapid-fire master Q&A](../06-interview/20-rapid-fire-master-qa.md)**
