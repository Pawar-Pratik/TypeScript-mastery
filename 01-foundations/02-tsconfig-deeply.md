# Lesson 02 — tsconfig, flag by flag

> **Why this lesson exists:** `tsconfig.json` decides how much TypeScript is actually *doing for you*. Two projects both "using TypeScript" can differ by an order of magnitude in bugs caught, entirely because of this file. Most engineers copy a config they don't understand and then wonder why a category of bug keeps shipping. **This is also the fastest way to look senior in a code review** — being able to say "turn on this flag, here's the bug class it catches" is immediately valuable.

**Time:** ~55 minutes · **Prereq:** Lesson 01

---

## 1. The idea in one sentence

> **`strict: true` is not one flag — it's a bundle of eight independent checks, and the two most valuable flags in TypeScript are *not* in it.**

---

## 2. The config I'd start every project with

```jsonc
{
  "compilerOptions": {
    /* ─── Language & environment ─── */
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],   // drop DOM for pure Node
    "module": "NodeNext",                        // or "ESNext" for a bundler
    "moduleResolution": "NodeNext",              // or "Bundler"

    /* ─── Type checking: the whole point ─── */
    "strict": true,                              // ← 8 flags, see §3
    "noUncheckedIndexedAccess": true,            // ★ not in strict. Turn it on
    "exactOptionalPropertyTypes": true,          // ★ not in strict. Turn it on
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "noPropertyAccessFromIndexSignature": true,
    "allowUnreachableCode": false,
    "allowUnusedLabels": false,
    /* Leave these to ESLint, not tsc — see §6 */
    // "noUnusedLocals": false,
    // "noUnusedParameters": false,

    /* ─── Interop & emit ─── */
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,                     // required by esbuild/swc/Vite
    "verbatimModuleSyntax": true,                // explicit `import type`
    "skipLibCheck": true,                        // see §5 — pragmatic, not lazy
    "resolveJsonModule": true,

    /* ─── Output ─── */
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true,
    "declaration": true,                         // libraries only
    "declarationMap": true,
    "noEmitOnError": true,                       // if tsc emits your output
    "incremental": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

Every line is justified below. **Don't copy it blind** — that's the habit this lesson exists to break.

---

## 3. `strict: true` — the eight flags inside it

Know each one and the bug class it catches. This is exam material.

### 1. `strictNullChecks` — **the most valuable flag in the language**
Without it, `null` and `undefined` are assignable to *every* type, so the compiler cannot help you with the single most common JavaScript error.

```ts
// strictNullChecks: false — the bad old world
function len(s: string) { return s.length; }
len(null);                     // ✅ compiles. 💥 at runtime

// strictNullChecks: true
function len(s: string) { return s.length; }
len(null);                     // ❌ Argument of type 'null' is not assignable to 'string'

function len2(s: string | null) {
  return s.length;             // ❌ 's' is possibly 'null'
}
function len3(s: string | null) {
  return s?.length ?? 0;       // ✅ you had to acknowledge it
}
```
Tony Hoare called null references his "billion-dollar mistake." **This flag is TypeScript's answer to it, and it's why the language is worth using.** If a codebase has this off, that's the finding — everything else is secondary.

### 2. `noImplicitAny`
```ts
function greet(name) { }       // ❌ Parameter 'name' implicitly has an 'any' type
function greet(name: string) { }  // ✅
```
Without it, every un-annotated parameter is `any`, and `any` spreads silently through everything it touches.

### 3. `strictFunctionTypes`
Checks function *parameter* types contravariantly — for function-type expressions, but **not for methods**. That exception is deliberate and is the subject of [Lesson 06](../02-type-system/06-structural-typing-and-variance.md); for now, note that it's why the `Array<Animal>`/`Array<Dog>` hole exists.

### 4. `strictBindCallApply`
Type-checks `.bind()`, `.call()` and `.apply()` arguments, which were previously `any[]`.

### 5. `strictPropertyInitialization`
```ts
class User {
  name: string;                // ❌ has no initializer and is not definitely assigned
  email!: string;              // ✅ "trust me" — used by DI frameworks
  role: string = "viewer";     // ✅
  constructor(public id: string) {}   // ✅ parameter property
}
```

### 6. `noImplicitThis`
```ts
function handler() { console.log(this.value); }   // ❌ 'this' implicitly has type 'any'
```

### 7. `alwaysStrict`
Emits `"use strict"` and parses in strict mode. Rarely noticed.

### 8. `useUnknownInCatchVariables`
```ts
try { risky(); }
catch (e) {
  console.log(e.message);           // ❌ 'e' is of type 'unknown'
}
// ✅ you must narrow, because JS can throw ANYTHING — including a string or null
catch (e) {
  const msg = e instanceof Error ? e.message : String(e);
}
```
**Correct and underappreciated:** `throw "oops"` is legal JavaScript, so `catch` genuinely receives `unknown`. This flag forces you to handle that. See [Lesson 16](../04-real-code/16-async-and-errors.md).

---

## 4. The two flags that *should* be in strict but aren't

These catch more real bugs per keystroke than anything else you can enable.

### `noUncheckedIndexedAccess` — **turn this on**

```ts
// OFF (the default) — TypeScript lies to you
const arr: string[] = [];
arr[0].toUpperCase();                    // ✅ compiles. 💥 TypeError at runtime

const map: Record<string, number> = {};
const n: number = map["missing"];         // ✅ compiles. n is undefined

// ON
const arr: string[] = [];
arr[0].toUpperCase();                    // ❌ 'arr[0]' is possibly 'undefined'

const first = arr[0];                     // string | undefined  ← the truth
if (first) first.toUpperCase();           // ✅
arr.at(0)?.toUpperCase();                 // ✅
const [head] = arr;                       // string | undefined too
```

**Why it's not in `strict`:** it's genuinely noisy in loop-heavy code, and it was added long after `strict` was established, so enabling it there would have broken millions of projects.

```ts
// The main annoyance, and the idiomatic answers:
for (let i = 0; i < arr.length; i++) {
  arr[i].trim();                   // ❌ possibly undefined — the compiler can't
}                                  //    correlate `i < arr.length` with `arr[i]`

for (const s of arr) s.trim();     // ✅ iteration is fine — prefer it anyway
arr.forEach(s => s.trim());        // ✅
const s = arr[i]; if (s) s.trim(); // ✅ when you need the index
```

**The verdict:** turn it on. Index access returning a possibly-missing value is *the truth*, and the flag mostly pushes you toward `for...of`, `.at()` and destructuring — which are better code anyway. It's also the one flag that fixes the classic `Record<string, T>` lie, where every lookup pretends to succeed.

### `exactOptionalPropertyTypes` — turn this on too

```ts
interface Options { timeout?: number }

// OFF: `timeout?: number` really means `number | undefined`
const a: Options = { timeout: undefined };    // ✅ allowed

// ON: optional means "may be ABSENT", not "may be undefined"
const b: Options = { timeout: undefined };    // ❌ not assignable
const c: Options = {};                        // ✅ absent
interface Options2 { timeout?: number | undefined }   // ✅ if you really mean both
```

**Why this matters far more than it looks:** it's exactly the PATCH problem from [API Lesson 09](../../API/02-rest-design/09-writes-patch-and-bulk.md). "Field absent" means *don't change it*; "field is `null`/`undefined`" means *clear it*. Those are different operations, and without this flag TypeScript can't tell them apart — so a bug where a user can't clear their phone number becomes invisible to the compiler.

```ts
// With the flag on, this distinction is expressible and enforced:
type PatchCustomer = {
  name?: string;                    // absent = unchanged
  phone?: string | null;            // absent = unchanged, null = CLEAR
};
```

---

## 5. Module resolution — where most config pain lives

The most confusing part of `tsconfig`, and the source of most "cannot find module" errors.

| `module` / `moduleResolution` | Use when |
|---|---|
| `"NodeNext"` / `"NodeNext"` | **Node.js**, respecting `package.json#type`. Requires file extensions in relative imports for ESM |
| `"ESNext"` / `"Bundler"` | **Vite/webpack/esbuild** — extensionless imports, `exports` map aware. Use this for front-end |
| `"CommonJS"` / `"Node10"` | Legacy Node. Doesn't understand `exports` maps — the cause of many `@types` failures |
| `"Preserve"` | TS 5.4+: leaves imports alone for a bundler to handle |

```jsonc
// Node ESM — extensions required, and it's the .js extension even in .ts files
{ "module": "NodeNext", "moduleResolution": "NodeNext" }
```
```ts
import { db } from "./lib/db.js";     // ← .js, not .ts. Surprises everyone once.
```
The reason: the emitted JavaScript imports a `.js` file, and TypeScript doesn't rewrite specifiers. It's the correct behaviour and it feels wrong the first time.

### The three interop flags

**`esModuleInterop: true`** — makes `import express from "express"` work with CommonJS modules by emitting the correct interop helpers. **Always on.** Without it you need `import * as express`, which is technically wrong for a callable export.

**`isolatedModules: true`** — forbids constructs that can't be understood one file at a time. **Required if you use esbuild/swc/Vite**, because they transpile each file independently with no type information.
```ts
export { SomeType };                  // ❌ isolatedModules: can't tell if it's a type
export type { SomeType };             // ✅
```

**`verbatimModuleSyntax: true`** — imports are emitted exactly as written, so you must mark type-only imports:
```ts
import type { User } from "./types";        // erased entirely
import { createUser } from "./users";       // kept
import { type User, createUser } from "./users";  // ✅ mixed, inline
```
Why it's worth it: it makes the emit predictable and prevents accidental runtime imports of type-only modules (which can trigger side effects or break tree-shaking). It replaces the older `importsNotUsedAsValues`/`preserveValueImports` pair.

### `skipLibCheck: true` — pragmatic, not lazy
Skips type-checking `.d.ts` files in `node_modules`. **Recommended**, and here's the honest reasoning: you cannot fix other people's type definitions, two dependencies with conflicting `@types/node` versions will produce hundreds of errors you have no power over, and it dramatically speeds up compilation. The cost is that a genuinely broken `.d.ts` in a dependency goes unnoticed — which you'd discover through use anyway.

> For a **library you publish**, run one CI job with `skipLibCheck: false` so you catch problems in your own emitted `.d.ts`.

---

## 6. `target` and `lib` — the distinction people get wrong

- **`target`** = which JavaScript *syntax* to emit. `ES2022` means the output keeps `async/await`, optional chaining, class fields.
- **`lib`** = which *type declarations* for built-ins are available. `ES2022` gives you `Array.prototype.at`, `Object.hasOwn`, `Error.cause`.

`lib` defaults from `target`, but they're independent — and that matters:

```jsonc
// Modern syntax, but I promise polyfills for the newer APIs:
{ "target": "ES2020", "lib": ["ES2022", "DOM"] }
```

**The trap:** `lib` is a *promise*, not a polyfill. Setting `"lib": ["ES2022"]` while targeting a browser that lacks `Array.at()` gives you a clean compile and a runtime crash. `lib` must match your actual runtime, or your polyfills must.

```jsonc
"lib": ["ES2022", "DOM", "DOM.Iterable"]   // browser
"lib": ["ES2022"]                            // Node — no DOM, so `document` is an error. Good.
```
Including `DOM` in a Node project is a small but real mistake: `document`, `window` and `fetch`'s DOM types become available and nothing stops you using them.

---

## 7. Flags I *don't* recommend

| Flag | Why not |
|---|---|
| `noUnusedLocals`, `noUnusedParameters` | Correct rules, wrong tool. They make `tsc` **fail** while you're mid-edit, which is maddening. Use ESLint's `@typescript-eslint/no-unused-vars` with `argsIgnorePattern: "^_"` — warnings while you work, errors in CI |
| `allowJs` + `checkJs` | Useful for **migration** ([L19](../05-ecosystem/19-migration-and-strictness.md)), noise otherwise |
| `strictNullChecks: false` | This is the single worst configuration choice available. If you inherit it, fixing it is the highest-value work in the repo |
| `suppressImplicitAnyIndexErrors` | Deprecated. Use `noPropertyAccessFromIndexSignature` and fix the access instead |
| `experimentalDecorators` | Only if a framework requires it (NestJS, TypeORM, older Angular). TS 5.0+ supports **standard** decorators without this flag; the two are not compatible |
| `baseUrl` + long `paths` maps | Works, but the bundler must be configured identically, and it usually is not. Prefer package.json `imports` (`#lib/*`) or a monorepo workspace |

---

## 8. `tsconfig` for different project types

```jsonc
// ── Node backend (Ledger API) ──
{ "compilerOptions": {
    "target": "ES2022", "lib": ["ES2022"],
    "module": "NodeNext", "moduleResolution": "NodeNext",
    "strict": true, "noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true,
    "esModuleInterop": true, "skipLibCheck": true, "verbatimModuleSyntax": true,
    "outDir": "dist", "rootDir": "src", "sourceMap": true, "noEmitOnError": true }}

// ── React app (Ledger Console) ──
{ "compilerOptions": {
    "target": "ES2022", "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext", "moduleResolution": "Bundler",
    "jsx": "react-jsx",                       // ← no `import React` needed
    "strict": true, "noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true,
    "noEmit": true,                            // Vite emits; tsc only checks
    "isolatedModules": true, "verbatimModuleSyntax": true, "skipLibCheck": true }}

// ── Published library ──
{ "compilerOptions": {
    "declaration": true, "declarationMap": true, "sourceMap": true,
    "strict": true, "noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true,
    "module": "NodeNext", "moduleResolution": "NodeNext",
    "outDir": "dist", "rootDir": "src", "noEmitOnError": true,
    "skipLibCheck": false }}                   // ← check your own .d.ts output
```

**Project references** (monorepos) get their own treatment in [Lesson 18](../05-ecosystem/18-tooling-and-performance.md), but the shape is:
```jsonc
// tsconfig.json at the root — a solution file with no files of its own
{ "files": [], "references": [{ "path": "./packages/shared" }, { "path": "./packages/api" }] }
```

---

## 9. Production rules

| Rule | Why |
|---|---|
| **`strict: true` from line one on any new project** | Retrofitting `strictNullChecks` into a mature codebase is a multi-week project |
| **`noUncheckedIndexedAccess: true`** | Array and record access genuinely can be `undefined`. This is the flag that stops lying to you |
| **`exactOptionalPropertyTypes: true`** | Distinguishes "absent" from "undefined" — the PATCH-semantics bug |
| **`lib` must match your real runtime** | It's a promise, not a polyfill |
| **No `DOM` lib in a Node project** | Or you'll use browser globals that don't exist |
| **`isolatedModules` + `verbatimModuleSyntax` if you use a bundler** | Prevents constructs your transpiler can't handle |
| **`skipLibCheck: true` in apps, `false` in one CI job for libraries** | You can't fix others' types; you must fix your own |
| **Unused-variable rules in ESLint, not `tsc`** | So a mid-edit variable doesn't fail your build |
| **One `tsconfig` per package; a root solution file with `references`** | Independent, cacheable checks |
| **Never `paths` aliases without matching bundler config** | The #1 cause of "works in tsc, fails at runtime" |

---

## 10. Interview traps

**Q1. "What does `strict: true` turn on?"**
Name as many of the eight as you can, and **lead with `strictNullChecks`** as the one that matters. Then the differentiator: *"and the two most valuable flags aren't in it — `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` — because they were added after `strict` was frozen."*

**Q2. "What's the most important compiler flag?"**
`strictNullChecks`. Without it, `null` and `undefined` are assignable to every type, so the compiler is silent about the most common runtime error in JavaScript. Hoare's billion-dollar mistake.

**Q3. "What does `noUncheckedIndexedAccess` change, and why is it off by default?"**
It makes index access return `T | undefined`, which is the truth. Off by default because it's noisy in index-based loops and would have broken existing projects when added. Turn it on; it pushes you toward `for...of`, `.at()` and destructuring.

**Q4. "`exactOptionalPropertyTypes` — what problem does it solve?"**
It separates "the key is absent" from "the key is present with value `undefined`". That distinction is the difference between "don't change this field" and "clear this field" — the PATCH-semantics bug, expressed in the type system.

**Q5. "Should you use `skipLibCheck`?"**
Yes for applications: you can't fix third-party `.d.ts` files, conflicting `@types` versions produce unfixable errors, and it's much faster. No for libraries in at least one CI job, so you validate your own emitted declarations.

**Q6. "`target` vs `lib`?"**
`target` = emitted syntax. `lib` = available built-in type declarations. Independent, and `lib` is a *promise* — declaring `ES2022` doesn't polyfill `Array.at`, so a mismatch with your real runtime is a clean compile and a runtime crash.

**Q7. "Why do I need `.js` extensions in my TypeScript imports?"**
With `module: NodeNext` and ESM, TypeScript doesn't rewrite specifiers — the emitted JS imports exactly what you wrote, and Node ESM requires extensions. So you write the extension of the *output* file. Correct, and counter-intuitive.

**Q8. "What's `isolatedModules` for?"**
It forbids constructs that can't be transpiled one file at a time — required by esbuild/swc/Vite, which have no cross-file type information. The classic offender is re-exporting a type without `export type`.

**Q9. "`any` is banned in your codebase. How do you enforce it?"**
Not via `tsconfig` — there's no flag. ESLint: `@typescript-eslint/no-explicit-any`, plus `no-unsafe-assignment`/`no-unsafe-member-access`/`no-unsafe-call`/`no-unsafe-return` from the `strict-type-checked` preset. **Those four unsafe-* rules matter more than banning the keyword**, because most `any` in a real codebase arrives implicitly from an untyped library rather than from someone typing the word.

**Q10. "How would you introduce strict mode into a large existing codebase?"**
Not all at once. Per-flag, per-directory, with a ratchet: enable one flag, fix or `// @ts-expect-error` the failures, lock it in CI, repeat. `strictNullChecks` last, because it's the biggest. Full strategy in [Lesson 19](../05-ecosystem/19-migration-and-strictness.md).

---

## 11. Build & break

### Build — feel each flag
In your `ts-lab`, write `src/flags.ts` containing code that violates each of the eight strict flags plus the two extras. Then toggle each flag in `tsconfig.json` one at a time and watch which errors appear and disappear.

```ts
// Each of these should error under exactly one flag. Verify which.
function a(x) { return x; }                            // noImplicitAny
function b(s: string) { return s.length; } b(null);     // strictNullChecks
class C { name: string; }                               // strictPropertyInitialization
function d() { return this.x; }                         // noImplicitThis
try { } catch (e) { e.message; }                        // useUnknownInCatchVariables
const arr: string[] = []; arr[0].trim();                // noUncheckedIndexedAccess
const o: { t?: number } = { t: undefined };             // exactOptionalPropertyTypes
function e(n: number): string { if (n > 0) return "x"; } // noImplicitReturns
switch (1 as number) { case 1: console.log("a"); case 2: break; }  // noFallthroughCasesInSwitch
```
**Predict the error before you toggle the flag.** That prediction is the exercise.

### Break — three configuration bugs to experience
1. **The `lib` lie.** Set `"target": "ES2015", "lib": ["ES2022"]`, use `[1,2,3].at(-1)`, compile cleanly, then run it in an environment without `Array.at`. Clean compile, runtime crash.
2. **The `strictNullChecks: false` world.** Turn it off in a file with 20 nullable accesses and watch every error vanish. Then turn it back on and count them. **That count is what the flag is worth.**
3. **The `paths` trap.** Add `"paths": { "@/*": ["src/*"] }` without configuring your bundler/runtime to match. `tsc --noEmit` passes; `node dist/index.js` throws `Cannot find module '@/lib/db'`.

### Build — Ledger's config
Write the real `tsconfig.json` for both Ledger (Node) and Ledger Console (React) using §8. For each non-obvious flag, add a **`//` comment saying why**. That commented config is a genuinely good thing to have in a portfolio repo — it shows you configured rather than copied.

### Explain out loud (60 seconds)
1. The eight flags in `strict`, leading with the important one.
2. The two flags that should be in it and aren't, and what each catches.
3. `target` vs `lib`, and the trap.
4. Why unused-variable checks belong in ESLint, not `tsc`.

---

## What's next

The compiler is configured. Now the type system itself — starting with the mental model that makes everything else derivable: **a type is a set of values**, and assignability is subset-hood. That single reframe explains unions, intersections, `never`, `unknown`, and every "why is this assignable?" question you'll ever have.

Next → **[Lesson 03: Types as sets](03-types-as-sets.md)**
