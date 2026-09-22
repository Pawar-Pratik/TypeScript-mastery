# Lesson 14 — Modules, declaration files & ambient types

> **Why this lesson exists:** *"Cannot find module 'x' or its corresponding type declarations"* is the single most common TypeScript error, and most engineers fix it by guessing — installing `@types/*`, adding `any`, or copying a `paths` config from Stack Overflow. This lesson gives you the resolution algorithm, so you diagnose instead of guessing, and teaches you to write `.d.ts` files, which is how you type an untyped dependency properly rather than surrendering to `any`.

**Time:** ~60 minutes · **Prereq:** Lessons 02, 13

---

## 1. The idea in one sentence

> **TypeScript resolves types by walking a defined lookup order for every import — and every module error is a step in that order failing, which means you can always find out *which* step rather than guessing.**

---

## 2. Module vs script — the rule that explains "duplicate identifier"

```ts
// utils.ts — NO import or export
const helper = 1;        // ← this is a GLOBAL. Visible everywhere.

// utils.ts — with any import or export
export const helper = 1; // ← module-scoped
```

**A file with no top-level `import` or `export` is a *script*, and everything in it is global.** That's why you sometimes get "Cannot redeclare block-scoped variable 'name'" between two unrelated files — they're both scripts declaring the same name in the global scope.

The fix, when a file genuinely has nothing to export:
```ts
export {};    // ← makes it a module. A common and useful line.
```

---

## 3. How TypeScript resolves a module's types

For `import { x } from "some-lib"`, with `moduleResolution: "NodeNext"` or `"Bundler"`:

```
1. Is it a relative path? ("./foo")
   → look for ./foo.ts, ./foo.tsx, ./foo.d.ts, ./foo/index.ts …
     (NodeNext + ESM: you must write ./foo.js — the OUTPUT name)

2. Is there a `paths` mapping in tsconfig? → apply it

3. node_modules/some-lib/package.json:
   a. "exports" field with a "types" condition  ← MODERN. Checked first
        "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.mjs" } }
   b. "types" or "typings" field
   c. "main" field → look for a sibling .d.ts (dist/index.js → dist/index.d.ts)

4. node_modules/@types/some-lib/  ← DefinitelyTyped

5. A local ambient declaration:  declare module "some-lib"

6. FAIL → "Cannot find module 'some-lib' or its corresponding type declarations"
```

### Diagnosing, properly
```bash
npx tsc --traceResolution | grep -A 20 "some-lib"
```
**`--traceResolution` prints every path TypeScript tried.** It's verbose and it's the definitive answer — use it instead of guessing. The output tells you exactly which of the six steps failed and what it looked for.

### The five causes, in order of frequency

| Symptom | Cause | Fix |
|---|---|---|
| "Cannot find module" for a JS-only package | No bundled types, no `@types` | Write a `.d.ts` (§4) or `npm i -D @types/x` |
| Works at runtime, fails in `tsc` | `moduleResolution` doesn't understand `exports` maps | Use `"NodeNext"` or `"Bundler"`, not `"Node10"` |
| "has no exported member" but it's clearly there | Dual CJS/ESM package with wrong condition resolution | Check the `exports` map; often needs `esModuleInterop` or `"Bundler"` |
| Works in `tsc`, fails at runtime | `paths` alias with no matching bundler/runtime config | Configure the bundler too, or use `package.json#imports` |
| Two copies of the same types | Duplicate versions in `node_modules` (e.g. two `@types/react`) | `npm ls @types/react`, then a resolution/override |

That third row deserves emphasis: **modern packages have `exports` maps, and `moduleResolution: "Node10"` (the old default) cannot read them.** If you're hitting inexplicable module errors on a modern dependency, this is usually it.

---

## 4. Writing `.d.ts` files

A declaration file describes shapes with **no implementation**. It's how you type JavaScript.

### Typing an untyped npm package
```ts
// src/types/untyped-lib.d.ts
declare module "untyped-lib" {
  export interface Options { retries?: number; timeout?: number }
  export function connect(url: string, opts?: Options): Promise<Connection>;
  export interface Connection {
    query<T = unknown>(sql: string, params?: unknown[]): Promise<T[]>;
    close(): Promise<void>;
  }
  const _default: { connect: typeof connect };
  export default _default;
}
```

```ts
// The escape hatch — only as a temporary step, and comment it
declare module "some-lib";        // implicitly `any`. Leaves a TODO you can grep for.
```

**The honest progression:** `declare module "x";` to unblock yourself today, then type the 5% of the API you actually use, then contribute it to DefinitelyTyped if the package is popular. Typing 5% is dramatically better than `any`, and much cheaper than typing all of it.

### Typing non-code imports
```ts
// src/types/assets.d.ts
declare module "*.svg" {
  import type { FC, SVGProps } from "react";
  const ReactComponent: FC<SVGProps<SVGSVGElement>>;
  export default ReactComponent;
}
declare module "*.css" { const classes: Record<string, string>; export default classes; }
declare module "*.module.css" { const classes: Record<string, string>; export default classes; }
declare module "*.json" { const value: unknown; export default value; }
```
> Vite and Next.js ship these (`vite/client`, `next/types`), so add them to `"types"` in tsconfig before writing your own.

### Augmenting existing modules
```ts
// Adding fields to Express's Request — the canonical use of declaration merging
import type { Principal, Repos } from "./domain";

declare global {
  namespace Express {
    interface Request {
      id: string;
      principal?: Principal;
      merchantId?: MerchantId;
      repos?: Repos;
    }
  }
}
export {};     // ← REQUIRED: makes this file a module so `declare global` is legal
```

```ts
// Augmenting a third-party module (not global)
declare module "fastify" {
  interface FastifyRequest { principal?: Principal }
}
```

**Two rules that catch everyone:**
1. `declare global` only works **inside a module** — so the file needs an `import` or `export {}`.
2. Module augmentation requires the module to already be imported somewhere in the program, or TypeScript treats your `declare module` as a *new* ambient declaration rather than an augmentation. The symptom is that your augmentation silently replaces the real types.

### Global variables and env vars
```ts
// src/types/globals.d.ts
declare global {
  interface Window { __LEDGER_CONFIG__?: { apiUrl: string; env: "live" | "test" } }

  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      REDIS_URL: string;
      NODE_ENV: "development" | "production" | "test";
      STRIPE_SECRET?: string;
    }
  }
}
export {};
```
⚠️ **Typing `ProcessEnv` is a lie you're choosing to tell.** It says `DATABASE_URL` is a `string` when it may be `undefined`. Better: validate env at boot with a schema and export the validated object ([Lesson 15](15-the-boundary-and-parsing.md)):
```ts
// ✅ the honest version — fails at startup, not at first request
export const env = EnvSchema.parse(process.env);
```

---

## 5. Import/export mechanics that matter

### `import type` and `verbatimModuleSyntax`
```ts
import type { Payment } from "./types";           // erased entirely
import { createPayment } from "./service";         // kept
import { type Payment, createPayment } from "./service";   // ✅ mixed, inline

export type { Payment };                            // required under isolatedModules
```
**Why it matters:** without `import type`, a transpiler working one file at a time can't tell whether an import is type-only, so it keeps the import — which can pull in a module purely for its types, triggering side effects and breaking tree-shaking. `verbatimModuleSyntax` makes the distinction mandatory and the emit predictable ([Lesson 02](../01-foundations/02-tsconfig-deeply.md)).

### Re-exports and barrel files
```ts
// src/domain/index.ts — a "barrel"
export * from "./payment";
export * from "./money";
export type { Customer } from "./customer";
```

**Barrels have a real cost, and this is worth knowing because it's a common performance surprise:**
- Importing one symbol from a barrel loads **every** module in it.
- They create circular-import risk, which in ESM produces `undefined` at runtime rather than an error.
- They slow the compiler, because every file importing the barrel depends on all of it.
- Tree-shaking helps in a bundler but not in Node, and not for side-effectful modules.

> **The rule:** barrels are fine for a package's *public* API (one at the root of a published library). Avoid them for internal folders — import from the specific file. Next.js and Vite both ship workarounds (`optimizePackageImports`) precisely because barrels hurt.

### Circular imports
```ts
// a.ts
import { b } from "./b";
export const a = () => b();
// b.ts
import { a } from "./a";
export const b = () => a();
```
Types are fine in a cycle (the checker resolves them). **Values are not** — at runtime one side sees `undefined`. Fixes: extract the shared piece into a third module, use `import type` where only the type is needed, or restructure so the dependency flows one way.

### `declare module` vs `namespace`
```ts
// ❌ Legacy — pre-dates ES modules. Don't write new code in namespaces.
namespace Ledger { export const version = "1"; }

// ✅ ES modules
export const version = "1";
```
`namespace` still appears in `.d.ts` files for global augmentation (`namespace Express`, `namespace NodeJS`) — that's its remaining legitimate use.

---

## 6. Publishing a typed library

If you publish anything to npm:

```jsonc
// package.json
{
  "name": "@ledger/sdk",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",      // ← MUST be first in each condition block
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist"],
  "sideEffects": false                     // enables tree-shaking
}
```

| Rule | Why |
|---|---|
| `"types"` **first** in each `exports` condition | Conditions are matched in order; types must win before `import`/`require` |
| Ship `.d.ts` **and** `.d.ts.map` | Consumers can go-to-definition into your source |
| `skipLibCheck: false` in one CI job | Validate your own emitted declarations |
| Run **[`arethetypeswrong`](https://arethetypeswrong.github.io)** in CI | Catches the dual CJS/ESM type problems that are otherwise invisible |
| Don't export types that reference private paths | Consumers get "cannot find module" for your internal files |
| Avoid `declare global` in a library | You're polluting every consumer's global scope |

`attw` (arethetypeswrong) deserves the callout: the CJS/ESM types story has several failure modes that only appear in consumer projects, and it's the tool that finds them.

---

## 7. Ledger's declaration layout

```
ledger/
├── src/
│   ├── types/
│   │   ├── env.d.ts            # NodeJS.ProcessEnv (and the note that validation is better)
│   │   ├── express.d.ts        # Request augmentation
│   │   └── assets.d.ts         # (console only) *.svg, *.css
│   ├── domain/
│   │   ├── payment.ts
│   │   ├── money.ts
│   │   └── index.ts            # a barrel — public surface only
│   └── lib/
└── tsconfig.json               # "include": ["src/**/*"]  ← must cover src/types
```

```ts
// src/types/express.d.ts
import type { Principal, Repos, MerchantId } from "../domain";

declare global {
  namespace Express {
    interface Request {
      id: string;
      principal?: Principal;
      merchantId?: MerchantId;
      repos?: Repos;
      rawBody?: Buffer;          // needed for HMAC webhook verification (API L23)
    }
  }
}
export {};
```

That `rawBody` field is a nice example of why augmentation matters: [API Lesson 23](../../API/05-beyond-rest/23-realtime-and-webhooks.md) requires verifying HMAC signatures against the **raw** bytes, so the middleware stashes them on the request — and the type system needs to know.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **`export {}` in any file that would otherwise be a script** | Prevents accidental globals and duplicate-identifier errors |
| **`--traceResolution` to diagnose module errors** | Definitive; stop guessing |
| **`moduleResolution: "NodeNext"` or `"Bundler"`, never `"Node10"`** | Node10 can't read `exports` maps |
| **Type 5% of an untyped library rather than `any`-ing it** | Cheap and dramatically better |
| **`declare module "x";` only as a temporary, commented step** | Leaves a greppable TODO |
| **Validate env with a schema instead of typing `ProcessEnv`** | Typing it is a lie that fails at first use, not at boot |
| **`import type` / `verbatimModuleSyntax`** | Predictable emit; no accidental runtime imports |
| **Barrels only at a package's public root** | Internal barrels load everything and risk cycles |
| **Break value cycles; type-only cycles are fine** | ESM cycles give `undefined` at runtime, not an error |
| **`"types"` first in every `exports` condition** | Conditions match in order |
| **Run `attw` in CI for published packages** | The dual CJS/ESM failures are invisible locally |
| **Make sure `include` covers `src/types`** | A `.d.ts` outside `include` is silently ignored — a classic head-scratcher |

---

## 9. Interview traps

**Q1. "Cannot find module 'x' or its corresponding type declarations — walk me through fixing it."**
Don't guess; give the algorithm: relative → `paths` → package `exports.types` → `types`/`typings` → sibling `.d.ts` → `@types/*` → local `declare module`. Then `tsc --traceResolution` to see which step failed. The usual causes are a JS-only package, or `moduleResolution: "Node10"` unable to read a modern `exports` map.

**Q2. "How do you type an untyped npm package?"**
`declare module "x" { ... }` in a `.d.ts` inside your `include`. Type only the surface you use. Temporary escape: `declare module "x";` (implicitly `any`), with a comment. Long term: contribute to DefinitelyTyped.

**Q3. "What makes a file a module vs a script?"**
Any top-level `import` or `export`. Without one, declarations are **global**, which causes cross-file duplicate-identifier errors. `export {}` is the fix.

**Q4. "How do you add a property to Express's `Request`?"**
Declaration merging: `declare global { namespace Express { interface Request { … } } }` plus `export {}` to make the file a module. This is the concrete reason `interface` still matters — a `type` alias can't merge.

**Q5. "What's `import type` for?"**
Marks an import as type-only so it's erased. Required by `isolatedModules`/`verbatimModuleSyntax` because single-file transpilers have no type information — and it prevents importing a module at runtime purely for its types, which breaks tree-shaking and can trigger side effects.

**Q6. "What's wrong with barrel files?"**
Importing one symbol loads every module in the barrel, they create circular-import risk, and they slow the compiler. Fine for a published package's root; avoid for internal folders. Both Next.js and Vite ship explicit optimisations for this, which tells you it's a real problem.

**Q7. "Circular imports — what breaks?"**
Types are fine; **values** are not. In ESM, one side sees `undefined` at runtime rather than erroring. Fix by extracting shared code, using `import type`, or making the dependency one-directional.

**Q8. "How do you publish a package with correct types?"**
`exports` map with `"types"` **first** in each condition, ship `.d.ts` + `.d.ts.map`, `sideEffects: false`, one CI job with `skipLibCheck: false`, and run `arethetypeswrong` to catch dual CJS/ESM problems.

**Q9. "Should you type `process.env`?"**
You can, but it's a lie — it declares variables as `string` that may be absent, so failures surface at first use rather than at startup. Better: parse `process.env` with a schema at boot and export the validated object, so a missing variable is a startup crash with a clear message.

**Q10. "My `.d.ts` file seems to be ignored."**
Almost always it's outside `include`/`files` in tsconfig, or the augmentation targets a module that isn't imported anywhere (so it creates a new ambient declaration instead of merging). Both are silent.

---

## 10. Build & break

### Build — Ledger's declarations
Create `src/types/express.d.ts`, `src/types/env.d.ts` and (for the Console) `src/types/assets.d.ts`. Verify each works — add `req.merchantId` in a middleware and confirm it's typed, then **deliberately move the file outside `include`** and watch the augmentation silently stop working. That failure mode is worth experiencing once.

### Build — type an untyped package
Find a small JS-only package you use. Write a `.d.ts` covering only the functions you call. Then compare: how many lines did it take, versus `any`? (Usually 10–20 lines for real safety.) If it's popular, open a DefinitelyTyped PR — that's a genuinely good thing to have in a portfolio.

### Break — five experiments
1. **Script vs module.** Create two files with `const x = 1` and no exports. Read the duplicate-identifier error. Add `export {}` to one.
2. **`--traceResolution`.** Run it for a package you have and one you don't. Read what it tried.
3. **Wrong `moduleResolution`.** Set `"Node10"` and import a package with an `exports` map. Watch it fail despite being installed.
4. **Value cycle.** Create `a.ts`/`b.ts` importing each other's values, call across the cycle, and observe the `undefined`. Then convert to `import type` and see it work.
5. **`paths` without a bundler.** Add `"@/*": ["src/*"]`, compile cleanly, then run the output with `node` and read the runtime failure.

### Explain out loud (60 seconds)
1. The module resolution order, and the tool that shows it.
2. What makes a file a module, and why it matters.
3. How you'd add a field to `Express.Request`.
4. The cost of barrel files.
5. Why typing `process.env` is a lie, and what to do instead.

---

## What's next

The most important practical lesson in the track. You've built precise internal types — now the boundary: how external data becomes trusted data, why `as` is never the answer, and how to derive your types **from** your validators so they can't drift.

Next → **[Lesson 15: The boundary — `unknown`, parsing, and Zod](15-the-boundary-and-parsing.md)**
