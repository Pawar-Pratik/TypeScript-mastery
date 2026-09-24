# Lesson 18 — Build tooling, monorepos & compiler performance

> **Why this lesson exists:** *"our TypeScript build takes 4 minutes"* is a real problem at real companies, and the fixes are specific and knowable. This lesson covers the modern toolchain (and why `tsc` usually isn't your bundler any more), project references for monorepos, and — the part that makes you useful — how to **diagnose** a slow compile rather than guess at it.

**Time:** ~60 minutes · **Prereq:** Lessons 02, 14

---

## 1. The idea in one sentence

> **Type checking and transpiling are separate jobs with wildly different costs — transpiling is milliseconds and parallelisable, type checking is seconds-to-minutes and inherently whole-program, so every modern toolchain splits them.**

---

## 2. The modern toolchain

```
        ┌──────────────── type check ────────────────┐
        │  tsc --noEmit           (seconds–minutes)   │  ← the slow, correct part
        │  runs in CI + a watch terminal locally      │
        └─────────────────────────────────────────────┘
        ┌──────────────── transpile ──────────────────┐
        │  esbuild / swc / Rolldown   (milliseconds)  │  ← strips types, NO checking
        │  runs in the dev server and the bundler     │
        └─────────────────────────────────────────────┘
```

| Tool | Job | Speed |
|---|---|---|
| **`tsc`** | Type check, emit `.d.ts`, emit JS | Slowest; the only one that checks |
| **esbuild** | Transpile + bundle | ~100× faster; **no type checking** |
| **swc** | Transpile (Rust) | Similar; used by Next.js, Jest via `@swc/jest` |
| **Vite** | Dev server + build (esbuild + Rollup) | The front-end default |
| **tsx / tsup** | Run/build TS in Node | esbuild-based |
| **Babel** | Transpile with plugins | Legacy for TS; still used for specific transforms |

**The consequence to internalise (and it catches everyone once):** your dev server happily runs code with type errors, because esbuild never looks at types. So:

```jsonc
{
  "scripts": {
    "dev": "vite",                                   // fast, unchecked
    "typecheck": "tsc --noEmit",
    "typecheck:watch": "tsc --noEmit --watch",       // ← run this in a second terminal
    "build": "tsc --noEmit && vite build",           // check, THEN bundle
    "lint": "eslint . --max-warnings 0"
  }
}
```
Or use `vite-plugin-checker` / Next.js's built-in checking to surface errors in the browser overlay.

> **When you still need `tsc` to emit:** publishing a library (you need `.d.ts` files — esbuild can't produce them; `tsup`/`rollup-plugin-dts` wrap `tsc` for this), and any project using `experimentalDecorators` with emitted metadata.

---

## 3. Monorepos and project references

The problem at scale: one giant `tsconfig` re-checks everything on every change.

```
ledger/
├── packages/
│   ├── shared/      # types, schemas, money, brands   ← the TypeScript-track output
│   ├── api/         # depends on shared
│   └── console/     # depends on shared
├── tsconfig.json            # solution file
└── tsconfig.base.json       # shared compiler options
```

```jsonc
// tsconfig.base.json — one place for the options
{
  "compilerOptions": {
    "target": "ES2022", "module": "NodeNext", "moduleResolution": "NodeNext",
    "strict": true, "noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true,
    "skipLibCheck": true, "esModuleInterop": true, "verbatimModuleSyntax": true,
    "composite": true,              // ★ required for project references
    "declaration": true,
    "declarationMap": true,         // ★ go-to-definition jumps to SOURCE, not .d.ts
    "incremental": true
  }
}

// packages/shared/tsconfig.json
{ "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist", "rootDir": "src" },
  "include": ["src/**/*"] }

// packages/api/tsconfig.json
{ "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist", "rootDir": "src" },
  "include": ["src/**/*"],
  "references": [{ "path": "../shared" }] }          // ★ declares the dependency

// tsconfig.json — a SOLUTION file with no files of its own
{ "files": [], "references": [
    { "path": "./packages/shared" },
    { "path": "./packages/api" },
    { "path": "./packages/console" }
] }
```

```bash
tsc --build            # builds only what changed, in dependency order
tsc --build --watch    # incremental watch across the whole graph
tsc --build --clean    # remove outputs
```

**What project references buy you:**
- **Incremental builds** — change `api`, and `shared` isn't re-checked.
- **Enforced boundaries** — `api` can't import `console`'s internals; the reference graph is the dependency graph.
- **Parallelism** — independent projects build concurrently.
- **`.tsbuildinfo` caching** — reusable in CI.

**The costs, stated honestly:** you must build dependencies before consumers (no more "just run tsc"), `composite: true` forces `declaration: true` (so every package emits `.d.ts`), and the setup is fiddly. `declarationMap: true` is not optional in practice — without it, go-to-definition lands you in a generated `.d.ts` instead of the source, which developers hate.

> **The modern alternative worth knowing:** many monorepos skip project references and use **Turborepo** or **Nx** to cache and parallelise plain `tsc --noEmit` per package. Simpler to set up, and the caching is often better. Project references win when you need `.d.ts` emit anyway (publishing packages) or want TypeScript itself to enforce the dependency graph.

---

## 4. Diagnosing a slow compile

**Measure before optimising.** These are the actual tools.

```bash
# 1. Where is the time going?
tsc --noEmit --diagnostics
#   Files: 1243        Lines: 482910      Identifiers: 521003
#   Check time: 18.4s  Total time: 24.1s

# 2. Extended, per-phase
tsc --noEmit --extendedDiagnostics
#   Parse / Bind / Check / Emit time, plus memory and instantiation counts

# 3. A flame graph of the compile
tsc --noEmit --generateTrace ./trace
npx @typescript/analyze-trace ./trace
#   → "Check file X took 4.2s"   "Type instantiation at Y took 3.1s"

# 4. What files are even included?
tsc --noEmit --listFiles | wc -l
tsc --noEmit --explainFiles | head -50      # WHY each file is included
```

**`--generateTrace` + `analyze-trace` is the tool that actually finds the culprit.** It names the file and often the exact type instantiation costing you seconds. Most "TypeScript is slow" complaints resolve to one or two pathological types.

### The seven causes, in rough order of frequency

| Cause | Symptom | Fix |
|---|---|---|
| **Too many files included** | `--listFiles` shows `node_modules` or `dist` | Tighten `include`/`exclude`; never include build output |
| **`skipLibCheck: false`** | Large "check" time in `.d.ts` files | Turn it on (§ [Lesson 02](../01-foundations/02-tsconfig-deeply.md)) |
| **Complex conditional/mapped types** | `analyze-trace` points at one type | Simplify; name intermediates; add explicit return type annotations |
| **Large intersections instead of `interface extends`** | Repeated type resolution | Use `interface extends` — it's cached by name ([Lesson 04](../01-foundations/04-objects-and-interfaces.md)) |
| **Inferred return types across module boundaries** | Slow "check" phase | **Annotate exported function return types** — this is the highest-value single fix |
| **Deep generic instantiation** | "Type instantiation is excessively deep" | Cap recursion; add variance annotations (`in`/`out`) |
| **Barrel files** | Every file depends on everything | Import from the specific module ([Lesson 14](../04-real-code/14-modules-and-declarations.md)) |

### The specific fixes, with reasons

```ts
// 1. Annotate exported return types — the biggest single win
export function buildQuery(opts: Opts) { /* 40 lines */ }        // ❌ inferred everywhere
export function buildQuery(opts: Opts): SqlFragment { /* ... */ } // ✅ resolved once
// Why: without an annotation, every importing file must re-infer the return type.

// 2. interface extends beats intersection
interface A extends B, C { x: string }        // ✅ named, cached
type A2 = B & C & { x: string };               // ❌ anonymous, re-resolved

// 3. Prefer type references to inline structural types in hot paths
function f(x: { a: string; b: number; c: boolean }) {}   // ❌ re-created at each call
interface Args { a: string; b: number; c: boolean }
function f2(x: Args) {}                                    // ✅

// 4. Variance annotations for large generic types (Lesson 06)
interface Store<in out T> { get(): T; set(v: T): void }    // short-circuits variance computation

// 5. Cap recursive types
type Path<T, D extends number = 5> = D extends 0 ? never : /* recurse with D-1 */;
```

```jsonc
// 6. Incremental builds — free, and most people leave it off
{ "compilerOptions": { "incremental": true, "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo" } }
```
Cache `.tsbuildinfo` in CI (GitHub Actions `actions/cache`) and repeat builds drop dramatically.

---

## 5. ESLint and formatting

```jsonc
// eslint.config.js (flat config)
import tseslint from "typescript-eslint";

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,       // ← type-aware rules: the valuable ones
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: { parserOptions: { projectService: true } },  // TS 5.x + v8: fast type info
    rules: {
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      "@typescript-eslint/no-unnecessary-condition": "warn",        // finds dead null checks
      "@typescript-eslint/consistent-type-imports": "error",
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
      "@typescript-eslint/no-explicit-any": "warn",
    },
  },
);
```

**Type-aware linting is where ESLint earns its place in a TypeScript project** — `no-floating-promises`, `no-misused-promises` and `no-unnecessary-condition` catch bugs the compiler can't. The cost is that ESLint now needs type information, which makes it slower; `projectService: true` (typescript-eslint v8) is dramatically faster than the old `project` option.

**Biome / oxlint** are Rust-based alternatives, 10–100× faster — but they **cannot do type-aware rules**, which are the ones worth having. The pragmatic split: Biome for formatting and syntactic rules on every save, typescript-eslint for type-aware rules in CI.

For formatting, **Prettier** remains the default; Biome's formatter is faster and nearly compatible. Either is fine — just don't let formatting be a code-review topic.

---

## 6. Testing setup

```jsonc
// Vitest — the path of least resistance for a TS project
// vitest.config.ts
export default defineConfig({
  test: { environment: "node", globals: true, typecheck: { enabled: true } },
});
```
Vitest uses Vite/esbuild, so it **doesn't type-check your tests by default** — same trade as the dev server. `typecheck.enabled` turns on `*.test-d.ts` checking, which is where your [Lesson 12](../03-type-level/12-utility-types.md) type tests live.

| Runner | TS handling |
|---|---|
| **Vitest** | esbuild; instant; optional type-check mode. **Default choice** |
| **Jest + `@swc/jest`** | Fast transpile, no checking |
| **Jest + `ts-jest`** | Type-checks during tests — slow, but catches errors |
| **`node --test` + `tsx`** | Zero-config, no framework |

And run `tsc --noEmit` over your test files in CI regardless, so tests aren't a type-checking blind spot.

---

## 7. CI pipeline

```yaml
jobs:
  check:
    steps:
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - uses: actions/cache@v4                  # ← the .tsbuildinfo cache
        with:
          path: node_modules/.cache
          key: ts-${{ hashFiles('**/package-lock.json') }}-${{ github.sha }}
          restore-keys: ts-${{ hashFiles('**/package-lock.json') }}-
      - run: npm run typecheck                  # tsc --noEmit (or --build)
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

**Order matters:** typecheck first — it's the fastest way to fail on the most common class of error, and there's no point linting code that doesn't compile.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Type check and transpile are separate; run both** | Your dev server doesn't check types |
| **`tsc --noEmit` as a required CI check, plus a local watch terminal** | Or you write hours of broken code |
| **`incremental: true` and cache `.tsbuildinfo` in CI** | Free speedup, commonly left off |
| **Measure with `--generateTrace` + `analyze-trace` before optimising** | Most slowness is one or two pathological types |
| **Annotate exported function return types** | The single highest-value compile-speed fix |
| **`interface extends` over large intersections** | Named types are cached; intersections are re-resolved |
| **`skipLibCheck: true` in apps** | You can't fix others' `.d.ts` |
| **Avoid internal barrel files** | Every importer depends on everything |
| **Project references when you need `.d.ts` emit or enforced boundaries; Turborepo/Nx otherwise** | Different tools for different problems |
| **`declarationMap: true` with `composite`** | Or go-to-definition lands in generated `.d.ts` |
| **Type-aware ESLint rules in CI; fast syntactic linting on save** | The valuable rules need type info and are slower |
| **Pin the TypeScript version and match your editor to it** | Minor versions add errors |

---

## 9. Interview traps

**Q1. "Your build takes 4 minutes. How do you find out why?"**
The method, not a guess: `--diagnostics` for the phase breakdown, `--listFiles`/`--explainFiles` to check what's included, then `--generateTrace` + `analyze-trace` for a flame graph that names the exact file or type instantiation. Then the usual culprits: too many files, `skipLibCheck` off, unannotated exported return types, complex conditional types, barrel files.

**Q2. "Why doesn't Vite/esbuild catch type errors?"**
They transpile file-by-file without type information — that's precisely why they're 100× faster. Type checking is inherently whole-program. So you run `tsc --noEmit` separately, in a watch terminal locally and as a CI gate.

**Q3. "What are project references for?"**
Incremental, ordered builds in a monorepo, with TypeScript itself enforcing the dependency graph. Requires `composite: true` (which forces `declaration`), and you should add `declarationMap` so go-to-definition reaches source. Mention the alternative: Turborepo/Nx caching plain per-package `tsc --noEmit` is simpler and often enough.

**Q4. "What's the single highest-value change for compile speed?"**
Annotating exported function return types. Without them, every importing file re-infers the return type; with them it's resolved once. It's also better API hygiene ([Lesson 05](../02-type-system/05-functions-and-overloads.md)).

**Q5. "Why is `interface extends` faster than an intersection?"**
An interface is a named, cached type reference; `A & B & C` creates an anonymous type that must be re-resolved at each use. The TypeScript team's own performance guidance says to prefer `extends`. Negligible at 50 files, measurable at 5,000.

**Q6. "`skipLibCheck` — isn't that hiding errors?"**
Yes, deliberately. You can't fix third-party `.d.ts` files, conflicting `@types` versions produce hundreds of unfixable errors, and checking them is slow. Apps: on. Published libraries: off in at least one CI job, so you validate your own emitted declarations.

**Q7. "Biome or ESLint?"**
Biome is far faster but **cannot do type-aware rules** — and `no-floating-promises` / `no-misused-promises` / `no-unnecessary-condition` are the rules worth having in a TypeScript project. Pragmatic split: Biome for format + syntactic rules on save, typescript-eslint for type-aware rules in CI.

**Q8. "Do your tests type-check?"**
Only if you make them. Vitest and `@swc/jest` transpile without checking. Run `tsc --noEmit` over test files in CI, and use `vitest --typecheck` for `.test-d.ts` type tests.

**Q9. "How do you stop a 'type instantiation is excessively deep' error?"**
Cap the recursion with a depth counter parameter, simplify the type, name intermediate types, or add variance annotations. Usually it means the type is doing too much — and in application code, that's a signal to simplify the model instead.

---

## 10. Build & break

### Build — Ledger's monorepo
Set up `packages/shared`, `packages/api`, `packages/console` with `tsconfig.base.json`, per-package configs, project references and a root solution file. Verify:
- `tsc --build` builds in dependency order
- Editing `shared` rebuilds consumers; editing `api` doesn't rebuild `shared`
- Go-to-definition from `api` into `shared` lands in **source** (that's `declarationMap`)
- `console` importing `api`'s internals fails

### Build — the performance lab
On any project with 200+ files:
```bash
npx tsc --noEmit --extendedDiagnostics          # baseline
npx tsc --noEmit --generateTrace ./trace && npx @typescript/analyze-trace ./trace
```
Then make one change at a time and re-measure:
1. Turn `skipLibCheck` off, then on.
2. Annotate return types on your 10 largest exported functions.
3. Replace an internal barrel import with a direct one.
4. Enable `incremental` and run twice.

**Write the numbers down.** A measured before/after is the difference between "I've heard barrels are slow" and "I cut our check time from 18s to 11s, and here's how I found it."

### Break — four experiments
1. **The unchecked dev server.** Introduce a real type error and run `npm run dev`. It works. Then `tsc --noEmit`.
2. **`include` too wide.** Add `dist` to `include` and watch file count and check time double.
3. **A pathological type.** Write a deeply recursive conditional type over a large object and watch check time spike in the trace.
4. **Editor/CI mismatch.** Set your editor to a different TypeScript version than the project's and find a rule that disagrees.

### Explain out loud (60 seconds)
1. Why transpiling and type checking are separate, and what that means for your dev loop.
2. The diagnostic sequence for a slow build.
3. Three specific speed fixes and why each works.
4. What project references buy and cost, and the alternative.

---

## What's next

The final ecosystem lesson: taking a real JavaScript codebase to strict TypeScript **without stopping feature work** — the ratchet strategy, the order to enable flags, and how to handle the libraries that fight you.

Next → **[Lesson 19: Migration & strictness strategy](19-migration-and-strictness.md)**
