# TypeScript: Zero → Absolute Authority

> The goal is not "I can add type annotations."
> The goal is: **you use the type system to make wrong code impossible to write** — and you can explain, mechanically, why any given error message appears.

---

## Why most TypeScript knowledge is shallow

Most people learn TypeScript as "JavaScript with annotations." They add `: string` where the editor complains, reach for `any` when it fights back, and `as` when it really fights back. It works, right up until:

- *"Why does this compile? I'm passing the wrong type."* (structural typing + bivariance)
- *"Why does `arr[0]` say `string` when the array is empty?"* (`noUncheckedIndexedAccess`)
- *"Why did my `if (user.role === 'admin')` check not narrow?"* (widened literal types)
- *"Why does this generic infer `string` instead of `'a' | 'b'`?"* (inference sites and `const` type parameters)
- *"Why does `Partial<T>` break my update function?"* (`exactOptionalPropertyTypes`, and `undefined` vs absent)
- *"My API returns valid JSON and my code still crashed."* (**types are erased; nothing was validated**)

Every one of those has a precise, mechanical answer. This course teaches the mechanisms, so error messages become information rather than obstacles.

### The five ideas everything else derives from

1. **Types are erased.** They exist only at compile time. **Nothing** is checked at runtime unless you write code to check it.
2. **The type system is structural, not nominal.** Compatibility is decided by shape, not by name or declaration.
3. **A type is a set of values.** Unions are set union, intersections are set intersection, `never` is the empty set, `unknown` is the universal set. Assignability is subset-hood.
4. **TypeScript is deliberately unsound in specific places.** It chose productivity over provable correctness. Knowing *where* is what separates competence from confusion.
5. **Inference is a fixed set of rules, not magic.** Once you know where inference happens and how widening works, you can predict every inferred type.

Internalise those five and TypeScript stops surprising you.

---

## What you'll be able to do at the end

- Read any type error and know *which rule* produced it
- Model a domain so that illegal states **cannot be constructed** — not merely caught
- Write generic functions and types that infer correctly, including conditional and mapped types
- Explain `any` vs `unknown` vs `never` vs `void`, and why `unknown` belongs at every boundary
- Implement `Partial`, `Pick`, `Omit`, `Exclude`, `ReturnType`, `Awaited` and friends **from scratch**
- Set up `tsconfig` for a real project and defend every flag
- Type a React codebase without `any` and without fighting the compiler
- Migrate a JavaScript codebase incrementally with a strictness ratchet
- Diagnose a slow build and a 10,000-line type error

---

## The project

You'll build the **shared type layer and API client for Ledger Console** — the typed contract between the Ledger API (from the [API track](../API/README.md)) and the React dashboard (the [React track](../React/README.md)).

That means: branded ID types the compiler can't confuse, a `Money` type that can't be added across currencies, a `Result` type that forces error handling, a discriminated union for every API state, a runtime-validated boundary with inferred types, and a fully-typed fetch client generated from OpenAPI.

**Why this and not exercises:** it's the layer that makes the React track pleasant, and it forces every type-level technique for a real reason.

---

## Setup

| Tool | Notes |
|---|---|
| **Node 20+** | |
| **TypeScript 5.x** | `npm i -D typescript`. Always use the project's version, never a global one |
| **VS Code** | Set **"Use Workspace Version"** for TypeScript (bottom-right of any `.ts` file). Version mismatches between the editor and the build are a real source of confusion |
| **`tsx`** | `npm i -D tsx` — run `.ts` directly during learning |
| **The TypeScript Playground** | [typescriptlang.org/play](https://www.typescriptlang.org/play) — **use it constantly.** Hover to see inferred types; that hover is the primary learning tool in this track |
| **`ts-reset`** *(optional, later)* | Fixes some standard-library type looseness. Lesson 19 |

> **The single most important habit in this track: hover over everything.** The inferred type on hover is the compiler telling you exactly what it concluded. People who become fluent in TypeScript are people who look at that constantly; people who stay stuck are people who only read the red squiggles.

---

## Table of contents

### Module 1 — Foundations
> Go slowly. Nearly everyone who finds TypeScript frustrating is missing something in this module.

| # | Lesson | What you'll be able to do |
|---|---|---|
| 01 | [Why TypeScript, and the compilation model](01-foundations/01-why-typescript-and-erasure.md) | Explain what `tsc` does and doesn't do, and why "it compiled" guarantees nothing at runtime |
| 02 | [tsconfig, flag by flag](01-foundations/02-tsconfig-deeply.md) | Configure a real project and defend every setting; know what each strict flag catches |
| 03 | [Types as sets: primitives, literals, unions, `any`/`unknown`/`never`](01-foundations/03-types-as-sets.md) | Reason about assignability as subset-hood; use `unknown` and `never` deliberately |
| 04 | [Objects, interfaces vs types, and the shape rules](01-foundations/04-objects-and-interfaces.md) | Model object types precisely; explain excess property checks and optional-vs-undefined |

### Module 2 — The type system properly
| # | Lesson | What you'll be able to do |
|---|---|---|
| 05 | [Functions, overloads & contextual typing](02-type-system/05-functions-and-overloads.md) | Type any function shape; know when to overload and when not to |
| 06 | [Structural typing, assignability & variance](02-type-system/06-structural-typing-and-variance.md) | **The most important lesson in the course.** Explain why unsafe code compiles |
| 07 | [Narrowing & control-flow analysis](02-type-system/07-narrowing-and-control-flow.md) | Make the compiler prove your invariants; write type predicates and exhaustive switches |
| 08 | [Generics, properly](02-type-system/08-generics-properly.md) | Write generics that infer correctly; debug "why did it infer that?" |

### Module 3 — Type-level programming
| # | Lesson | What you'll be able to do |
|---|---|---|
| 09 | [Making illegal states unrepresentable](03-type-level/09-illegal-states-unrepresentable.md) | Design domains with discriminated unions, branded types and `Result` |
| 10 | [Conditional types & `infer`](03-type-level/10-conditional-types-and-infer.md) | Compute types from types; read any library's type gymnastics |
| 11 | [Mapped types & template literal types](03-type-level/11-mapped-and-template-literal-types.md) | Transform object types; build type-safe string APIs |
| 12 | [The utility types, and building your own](03-type-level/12-utility-types.md) | Implement every built-in from scratch; know when to stop |

### Module 4 — TypeScript in real code
| # | Lesson | What you'll be able to do |
|---|---|---|
| 13 | [Classes, `this`, and OOP in TypeScript](04-real-code/13-classes-and-oop.md) | Use classes where they earn their place; explain `private` vs `#private` |
| 14 | [Modules, declaration files & ambient types](04-real-code/14-modules-and-declarations.md) | Fix any "cannot find module" or `@types` problem; write `.d.ts` for untyped code |
| 15 | [The boundary: `unknown`, parsing, and Zod](04-real-code/15-the-boundary-and-parsing.md) | Never trust external data again; infer types *from* validators |
| 16 | [Async, errors & the `Result` pattern](04-real-code/16-async-and-errors.md) | Type promises precisely; handle errors in a way the compiler enforces |

### Module 5 — Ecosystem & scale
| # | Lesson | What you'll be able to do |
|---|---|---|
| 17 | [TypeScript with React](05-ecosystem/17-typescript-with-react.md) | Type components, hooks, events, generics and context without `any` |
| 18 | [Build tooling, monorepos & compiler performance](05-ecosystem/18-tooling-and-performance.md) | Diagnose a slow build; set up project references correctly |
| 19 | [Migration & strictness strategy](05-ecosystem/19-migration-and-strictness.md) | Move a real JS codebase to strict TS without stopping feature work |

### Module 6 — Interview mastery
| # | Lesson | What you'll be able to do |
|---|---|---|
| 20 | [Rapid-fire master Q&A](06-interview/20-rapid-fire-master-qa.md) | Revise the whole track; drill until answers are reflexes |
| 21 | [Type challenges](06-interview/21-type-challenges.md) | Solve live type puzzles — a real interview format |
| 22 | [Code review & design judgement](06-interview/22-review-and-judgement.md) | Review TypeScript like a staff engineer |

---

## The 15 questions that define mastery

Self-test now, note your score, retest at the end.

1. What does `tsc` do at runtime? (Trick question — answer precisely.)
2. `any` vs `unknown` vs `never` vs `void` — one sentence each, plus when you'd use each.
3. Why does this compile, and is it safe?
   ```ts
   const handlers: Array<(e: Event) => void> = [];
   const onClick = (e: MouseEvent) => console.log(e.clientX);
   handlers.push(onClick);
   ```
4. `interface` vs `type` — three real differences and when it matters.
5. Why does `const x = 'a'` give `'a'` but `let x = 'a'` give `string`?
6. What is an excess property check, and why does assigning through a variable bypass it?
7. What does `noUncheckedIndexedAccess` change, and why is it off by default?
8. Write a type predicate. Then write an assertion function. When do you need each?
9. How do you make a `switch` exhaustive so adding a union member is a compile error?
10. Implement `Omit<T, K>` from scratch. Then explain why it doesn't distribute over unions.
11. What does `infer` do? Implement `ReturnType<T>`.
12. What's a branded type and what problem does it solve?
13. Your API returns `{ id: number }` but the type says `{ id: string }`. When do you find out?
14. `enum` vs union of string literals vs `as const` object — which and why?
15. Your build takes 90 seconds. How do you find out why?

Every one is answered in the track, with the lesson named in [Lesson 20](06-interview/20-rapid-fire-master-qa.md).

---

## Pacing

| Intensity | Duration |
|---|---|
| **Sprint** (4 h/day) | ~2 weeks |
| **Serious** (2 h/day) | ~4 weeks — the sweet spot |
| **Steady** (1 h/day) | ~8 weeks |

Spend the most time on **Module 2** (structural typing, narrowing, generics). Everything in Modules 3–5 is an application of those three.

---

## One promise

Every "weird" TypeScript behaviour has a rule behind it. By the end you'll know the rules, so the compiler becomes a collaborator you can argue with — and win.

Start here → **[Lesson 01: Why TypeScript, and the compilation model](01-foundations/01-why-typescript-and-erasure.md)**
