# Lesson 01 — Why TypeScript, and the compilation model

> **Why this lesson exists:** almost every serious TypeScript misunderstanding traces back to one thing — **not knowing that types vanish at runtime.** Engineers write `user: User`, the API returns something else, and they're genuinely surprised the code crashed. This lesson makes the compilation model concrete, so "it compiled" never again feels like a guarantee it isn't.

**Time:** ~45 minutes · **Prereq:** working JavaScript knowledge

---

## 1. The idea in one sentence

> **TypeScript is a *static analysis tool* that reads your annotations, checks them for contradictions, and then deletes them — producing JavaScript that behaves exactly as if you'd never typed anything.**

Every word matters. Static: compile time only. Analysis: it reasons, it doesn't enforce. Deletes: nothing survives.

---

## 2. What TypeScript actually is: three separate jobs

People say "TypeScript" to mean three different tools that ship together.

| Job | What it does | Could you skip it? |
|---|---|---|
| **1. Type checker** | Reads annotations, reports contradictions | **This is the value.** Everything else is incidental |
| **2. Transpiler** | Strips types, optionally downlevels syntax (`ES2022 → ES5`) | Yes — esbuild, swc and Babel do it faster |
| **3. Language service** | Powers editor autocomplete, refactoring, go-to-definition | Enormous value, and it's the same engine |

The important consequence: **most modern build setups don't use `tsc` to produce JavaScript at all.** Vite/esbuild strips types in milliseconds without type-checking, and `tsc --noEmit` runs separately as a check.

```jsonc
// package.json — the modern split, and worth understanding
{
  "scripts": {
    "dev":       "vite",                 // esbuild strips types. NO type checking
    "build":     "tsc --noEmit && vite build",   // check, then bundle
    "typecheck": "tsc --noEmit"          // ← the actual type safety, in CI
  }
}
```

> **The trap this creates:** in dev, esbuild happily strips types from code that doesn't type-check, so **your app runs with type errors in it.** People discover this when CI fails on code that "worked all afternoon." Run `tsc --noEmit --watch` in a second terminal, or use `vite-plugin-checker`. Knowing why dev doesn't catch errors is a genuinely useful piece of knowledge.

---

## 3. Type erasure, made concrete

This is the mechanism. See it once and you'll never forget it.

```ts
// input.ts
interface User {
  id: string;
  email: string;
  role: "admin" | "viewer";
}

type Maybe<T> = T | null;

function greet(user: User, times: number = 1): string {
  return `Hello ${user.email}`.repeat(times);
}

const admin = { id: "u_1", email: "a@x.dev", role: "admin" } as User;
enum Status { Active, Inactive }          // ← the exception, see below
```

```js
// output.js — this is ALL of it
"use strict";
function greet(user, times = 1) {
  return `Hello ${user.email}`.repeat(times);
}
const admin = { id: "u_1", email: "a@x.dev", role: "admin" };
var Status;                                // ← enum survived!
(function (Status) {
  Status[Status["Active"] = 0] = "Active";
  Status[Status["Inactive"] = 1] = "Inactive";
})(Status || (Status = {}));
```

**Gone entirely:** `interface`, `type`, all annotations, all generics, `as`, `satisfies`, `readonly`, `private`, `implements`, non-null `!`.

**Survives, because it generates real code:** `enum` (a runtime object), `class` (a real JS class), and parameter default values.

### What this means, concretely

**1. You cannot check a type at runtime.**
```ts
// ❌ Does not compile — and even conceptually, `User` doesn't exist at runtime
if (value instanceof User) { }
if (typeof value === "User") { }

// ✅ You must check the SHAPE yourself
function isUser(v: unknown): v is User {
  return typeof v === "object" && v !== null &&
    "id" in v && typeof (v as any).id === "string" &&
    "email" in v && typeof (v as any).email === "string";
}
```
`instanceof` works on **classes** (which exist at runtime), never on interfaces or type aliases.

**2. No type-based dispatch, no reflection.**
```ts
// ❌ Impossible — the type argument doesn't exist at runtime
function make<T>(): T { return new T(); }
function fields<T>(): string[] { return Object.keys(T); }
```
Java generics are also erased; C# generics are *reified* (available at runtime). TypeScript is the Java model, taken further. This is why libraries need you to pass a schema or a class explicitly — the type alone can't be inspected.

**3. External data is unchecked, always.**
```ts
const user: User = await fetch("/api/me").then(r => r.json());
//                 ^^^^ this is a LIE the compiler happily believes
```
`response.json()` returns `Promise<any>`, and `any` assigns to anything. **No validation happened.** If the API returns `{ id: 42 }`, `user.id` is a number, `user.id.toUpperCase()` throws at runtime, and the compiler told you everything was fine.

> **This is the single most important practical consequence in the entire language**, and it's why [Lesson 15](../04-real-code/15-the-boundary-and-parsing.md) exists. It's also the direct link to the API track: types are your *internal* contract; the API's contract must be **validated** at the boundary, because nothing enforces it for you.

**4. Type errors don't stop compilation** (by default).
```bash
tsc                    # reports errors AND still emits .js
tsc --noEmitOnError    # emit nothing if there are errors
```
`tsc` is a linter that also transpiles. This surprises people from compiled-language backgrounds.

---

## 4. Why it exists: what the type checker buys you

Not "fewer bugs" in the abstract. Four specific, measurable things.

### 1. Errors move from runtime to compile time
```ts
function applyDiscount(price: number, pct: number) { return price * (1 - pct / 100); }
applyDiscount("49.99", 10);   // ❌ compile error
// In JS: "49.99" * 0.9 → 44.991 — works by coercion, until it doesn't
```
Microsoft's own analysis of public JS bugs suggested ~15% would have been caught by types. Not a silver bullet; a large, cheap fraction.

### 2. Refactoring becomes mechanical
Rename `email` → `emailAddress` on `User` and every one of 400 usages is a compile error. In JavaScript that's a project-wide text search and a prayer. **This is the benefit that grows superlinearly with codebase size**, and it's the real reason large teams adopt TypeScript.

### 3. The editor becomes a source of truth
Autocomplete, inline documentation, go-to-definition, safe automated refactors. That's the language service, and it's why even JS-only projects benefit from `// @ts-check` and JSDoc.

### 4. Types are documentation that can't rot
```ts
// A comment that will eventually be a lie:
/** @param opts - the options. `retries` defaults to 3 */
function fetchWithRetry(url, opts) { }

// A signature that cannot lie:
function fetchWithRetry(url: string, opts: { retries?: number; timeoutMs?: number }): Promise<Response>
```
That's exactly the layer-1 contract idea from [API Lesson 01](../../API/01-foundations/01-what-an-api-really-is.md) — and note the same limitation applies: the type says `timeoutMs: number`, but only prose can say *"milliseconds, default 5000, must be under the gateway's 25s."* **Types capture syntax; semantics still need words.**

### And what it costs — be honest about this

| Cost | Reality |
|---|---|
| Build step and toolchain complexity | Real, but you already have one |
| Learning curve for advanced types | Real. Modules 3 of this track exists because of it |
| **False confidence** | **The biggest one.** "It compiles" feels like "it's correct," and it isn't |
| Time fighting the compiler | Usually a sign you're modelling the domain wrong, not that TS is wrong |
| Some patterns are hard to type | Heavy metaprogramming, dynamic property access. `any` exists as an escape hatch for a reason |

---

## 5. TypeScript is deliberately unsound — and that's a design choice

A **sound** type system guarantees that if it type-checks, no type errors occur at runtime. TypeScript explicitly **is not sound**, and the team says so in their non-goals: they prioritise productivity and JavaScript compatibility over provable correctness.

The five deliberate holes — know all of them, because interviewers ask:

```ts
// 1. `any` — opts out entirely, and spreads silently
const x: any = "str";
const n: number = x;              // no error. `any` assigns to everything

// 2. Type assertions — you overriding the compiler
const el = document.getElementById("x") as HTMLInputElement;   // might be null, might be a div
JSON.parse(s) as User;                                          // pure hope

// 3. Array index access (without noUncheckedIndexedAccess)
const arr: string[] = [];
arr[0].toUpperCase();             // type says string. Runtime: TypeError

// 4. Method parameter bivariance (Lesson 06)
const dogs: Dog[] = [];
const animals: Animal[] = dogs;   // allowed
animals.push(new Cat());          // ← now dogs contains a Cat

// 5. External data — no runtime validation, ever
const u: User = await res.json();
```

**Why they accepted these:** each one has a large productivity payoff and the alternative would break JavaScript idioms. Array access is the clearest example — requiring an `undefined` check on every index would make normal loops unbearable, so it's opt-in via a flag ([Lesson 02](02-tsconfig-deeply.md)).

> **The framing to use in an interview:** *"TypeScript trades soundness for adoptability. It's a very good bug-finder, not a proof system. So I use the type system to eliminate whole categories of mistake, and I put runtime validation at every boundary where data enters from outside my program — because that's exactly where the type system's guarantees stop."*

---

## 6. Where TypeScript's guarantees begin and end

Draw this mentally. It's the map for everything else in the track.

```
        OUTSIDE (untrusted — types are claims, nothing is checked)
   ┌───────────────────────────────────────────────────────────┐
   │  HTTP responses · JSON.parse · localStorage · env vars    │
   │  URL params · form input · WebSocket messages · CSV       │
   │  Untyped npm packages · `any` from a bad @types           │
   └────────────────────────┬──────────────────────────────────┘
                            │  ← VALIDATE HERE. This line is the whole job.
                            │     unknown → parse → typed  (Lesson 15)
   ┌────────────────────────▼──────────────────────────────────┐
   │  INSIDE (the compiler's guarantees hold)                   │
   │  Your functions, your domain model, your components        │
   │  Refactors are safe. Exhaustiveness works. Types are real  │
   └───────────────────────────────────────────────────────────┘
```

Everything TypeScript promises is true **inside** that box. The single most valuable habit you can build is treating the boundary as a real, explicit thing: **`unknown` in, parse, typed out.**

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Run `tsc --noEmit` in CI as a required check** | Your dev server doesn't type-check |
| **Also run it in watch mode locally** | Or you'll write hours of code that doesn't compile |
| **Pin TypeScript in `devDependencies`; never use a global install** | Minor versions add errors; the build must be reproducible |
| **Set the editor to the workspace TypeScript version** | Otherwise your editor and CI disagree |
| **`strict: true` from day one** on new projects | Retrofitting is far more expensive ([L02](02-tsconfig-deeply.md)) |
| **Treat every external input as `unknown` and parse it** | The boundary is where guarantees stop |
| **Never `as` to make an error go away** | You've turned a compile error into a runtime one |
| **`--noEmitOnError` if `tsc` produces your output** | Don't ship code that failed the check |
| **Never say "it compiles, so it's correct"** | It's a bug-finder, not a proof system |

---

## 8. Interview traps

**Q1. "What does TypeScript do at runtime?"**
**Nothing.** All types are erased. The output is JavaScript that behaves identically to the untyped version. Then add the exceptions that *do* emit code: `enum`, `class`, parameter defaults, and (with `experimentalDecorators`) decorators. Volunteering that list shows you've actually looked at the output.

**Q2. "So what's the point if there's no runtime checking?"**
Four things: errors caught at compile time, mechanically safe refactoring at scale, editor tooling, and documentation that can't rot. Then the honest boundary: *"and that's exactly why I validate at every I/O boundary — the type system's job ends where my program's inputs begin."*

**Q3. "Your API returns `{ id: number }` and your type says `{ id: string }`. When do you find out?"**
**At runtime**, when something calls a string method on a number — possibly far from the fetch. `res.json()` is `any`/`Promise<any>`, which assigns to anything with no complaint. The fix is runtime parsing at the boundary, with the type *derived from* the validator so they can't diverge ([L15](../04-real-code/15-the-boundary-and-parsing.md)).

**Q4. "Is TypeScript sound?"**
No, deliberately. Name the five holes: `any`, assertions, array index access, method parameter bivariance, and unvalidated external data. Then the reasoning: soundness was traded for JavaScript compatibility and productivity, and each hole buys something concrete.

**Q5. "Can you check an interface at runtime?"**
No — interfaces don't exist at runtime. You check the *shape* with a type predicate, or better, use a schema library and derive the type from the schema. `instanceof` works only for classes.

**Q6. "Why does my dev server run code that has type errors?"**
Because esbuild/swc/Babel **strip** types without checking them — that's why they're 100× faster than `tsc`. Type checking is a separate process (`tsc --noEmit`). Fix: run it in watch mode locally and as a required CI check.

**Q7. "Does TypeScript make your code slower?"**
No — the emitted JavaScript is the same, modulo `enum` and downlevelling. The *build* is slower. The one real runtime cost is if you target old JavaScript and TS emits polyfills/helpers (e.g. downlevelled `async/await` becomes a state machine).

**Q8. "TypeScript or JSDoc-typed JavaScript?"**
A good question because it tests whether you know the same checker powers both. `// @ts-check` plus JSDoc gives you real type checking with **no build step** — genuinely right for small libraries and Node scripts, and the reason some projects (notably Svelte, which moved off `.ts` files) chose it. TypeScript syntax wins for expressiveness and ergonomics at scale. **Node's native type-stripping** (20+ with a flag, on by default in 23+) narrows the gap further by removing the build step for plain annotations.

---

## 9. Build & break

### Build — watch erasure happen
```bash
mkdir ts-lab && cd ts-lab && npm init -y && npm i -D typescript tsx
npx tsc --init
```
Write `src/erasure.ts` with an interface, a type alias, a generic function, an enum, a class, and a `satisfies`. Then:
```bash
npx tsc src/erasure.ts --outDir dist --target ES2022 --module ESNext
cat dist/erasure.js
```
**Diff them side by side.** Note exactly what disappeared and what didn't. This 5-minute exercise removes 80% of future confusion about TypeScript.

### Break — five experiments, all of which "compile fine"
```ts
// 1. Lie about external data
const user: { id: string } = JSON.parse('{"id": 42}');
console.log(user.id.toUpperCase());       // 💥 runtime TypeError. Zero compile errors.

// 2. Lie with an assertion
const el = document.getElementById("nope") as HTMLInputElement;
console.log(el.value);                     // 💥 Cannot read properties of null

// 3. Empty array indexing
const names: string[] = [];
console.log(names[0].length);              // 💥 — type said string

// 4. any as a silent infection
function parse(raw: any) { return raw.data.items.map((i: any) => i.name); }
const result: number = parse("{}");        // no error anywhere. All of it wrong.

// 5. Types don't exist at runtime
interface Cfg { url: string }
// console.log(Object.keys(Cfg));           // ❌ 'Cfg' only refers to a type
```
**Run every one.** Watching code that type-checks throw a `TypeError` is the lesson.

### Build — prove the dev-server gap
1. Create a Vite + TS project (`npm create vite@latest -- --template vanilla-ts`).
2. Introduce a real type error: `const n: number = "hello";`
3. Run `npm run dev`. **The app runs.** The browser shows no error.
4. Run `npx tsc --noEmit`. There's your error.
5. Add `vite-plugin-checker` (or a second terminal running `tsc --noEmit --watch`) and see it surface.

### Explain out loud (60 seconds)
1. What TypeScript does at runtime, and the exceptions that emit code.
2. The four things the type checker buys you, and the one big cost.
3. The five soundness holes.
4. Where the guarantees begin and end — draw the boundary diagram.

---

## What's next

You know what the compiler is. Next: how to configure it — because **`strict: true` is not one flag, it's eight**, and the difference between a project that catches real bugs and one that catches typos is almost entirely in `tsconfig.json`.

Next → **[Lesson 02: tsconfig, flag by flag](02-tsconfig-deeply.md)**
