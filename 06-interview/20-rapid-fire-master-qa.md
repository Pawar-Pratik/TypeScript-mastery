# Lesson 20 — Rapid-fire master Q&A

> **How to use this:** cover the answer column, say your answer **out loud**, then compare. Anything you hedge on goes into `RECALL.md` and gets re-tested cold every Sunday. Answers here are compressed to the *shape* of a strong answer — the lesson reference has the depth.

**Time:** ~90 min for a full pass · ~25 min to re-test

---

## Scoring

| Level | Sounds like |
|---|---|
| **1 — Recall** | The definition |
| **2 — Mechanism** | *Why* it works that way, and what breaks otherwise |
| **3 — Judgement** | Level 2 + the trade-off + what you'd do + what would change your mind |

---

## Part 1 — Foundations (25)

| # | Question | Strong answer |
|---|---|---|
| 1 | What does TypeScript do at runtime? | **Nothing.** All types are erased; the output behaves identically to untyped JS. Exceptions that *do* emit code: `enum`, `class`, parameter defaults, legacy decorators. [L01] |
| 2 | So what's the point? | Compile-time error detection, mechanically safe refactoring at scale, editor tooling, and documentation that can't rot. Then the boundary: *"and that's why I validate every external input — the guarantees end where my inputs begin."* [L01] |
| 3 | Is TypeScript sound? | No, deliberately. Five holes: `any`, type assertions, array index access, method-parameter bivariance, unvalidated external data. Traded soundness for JS compatibility and productivity. [L01] |
| 4 | Your API returns `{id: number}`, your type says `string`. When do you find out? | At runtime, far from the fetch. `res.json()` is `any`, which assigns silently. Fix: parse at the boundary and derive the type from the schema. [L01, L15] |
| 5 | Why does your dev server run code with type errors? | esbuild/swc **strip** types without checking — that's why they're 100× faster. Type checking is whole-program and separate (`tsc --noEmit`). [L01, L18] |
| 6 | Can you check an interface at runtime? | No — interfaces don't exist at runtime. Check the shape with a predicate, or better, use a schema and derive the type. `instanceof` works only for classes. [L01] |
| 7 | What's in `strict`? | Eight flags. Lead with `strictNullChecks`. Then: `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitThis`, `alwaysStrict`, `useUnknownInCatchVariables`. [L02] |
| 8 | Most important flag? | `strictNullChecks`. Without it `null`/`undefined` are assignable to every type, so the compiler is silent about the most common JS runtime error. [L02] |
| 9 | Which valuable flags are *not* in strict? | `noUncheckedIndexedAccess` (index access returns `T \| undefined` — the truth) and `exactOptionalPropertyTypes` (absent ≠ undefined). Both added after `strict` was frozen. [L02] |
| 10 | What does `exactOptionalPropertyTypes` solve? | It separates "key absent" from "key present with `undefined`" — which is the difference between "don't change this field" and "clear this field". The PATCH-semantics bug, in the type system. [L02] |
| 11 | `target` vs `lib`? | `target` = emitted syntax; `lib` = available built-in type declarations. Independent — and `lib` is a **promise, not a polyfill**, so a mismatch is a clean compile and a runtime crash. [L02] |
| 12 | `skipLibCheck` — hiding errors? | Yes, deliberately. You can't fix third-party `.d.ts`, conflicting `@types` produce unfixable errors, and it's much faster. On for apps; off in one CI job for published libraries. [L02] |
| 13 | Why `.js` extensions in TS imports? | With `module: NodeNext` + ESM, TypeScript doesn't rewrite specifiers — the emitted JS imports exactly what you wrote, and Node ESM requires extensions. So you write the *output* file's extension. [L02] |
| 14 | What is a type? | A **set of values**. "Assignable" means "subset of". Unions are set union, intersections set intersection, `never` the empty set, `unknown` the universal set. [L03] |
| 15 | `any` vs `unknown` vs `never` vs `void`? | `any` opts out (assignable both ways). `unknown` = every value (from everything, to nothing). `never` = no values (to everything, vacuously). `void` = return value unusable. Practical: `unknown` at boundaries, `never` for exhaustiveness, `any` last resort. [L03] |
| 16 | Why is `never` assignable to everything? | Vacuous truth — the assignability rule quantifies over the source set's values, and the empty set has none to violate it. Bottom type. [L03] |
| 17 | Why does `const x = 'a'` give `'a'` but `let` gives `string`? | Widening: `let` can be reassigned so TS widens to the base type. And **properties of a `const` object still widen** unless you use `as const`. [L03] |
| 18 | What does `as const` do? | Prevents literal widening and makes everything deeply `readonly`. Best use: `const X = [...] as const; type T = typeof X[number]` — one declaration gives a runtime array *and* a precise union. [L03] |
| 19 | `enum` or string literal union? | Union by default. `enum` **emits runtime code** (the only non-erased construct) and is **nominally typed**, so you can't pass `"pending"` where `Status.Pending` is expected — friction at every JSON boundary. `as const` object when you need the values at runtime. [L03] |
| 20 | How do you make a switch exhaustive? | `default: assertNever(x)` where `assertNever(x: never): never`. A missing variant means `x` isn't `never` → compile error. **This is what makes adding an API enum value safe.** [L03, L07] |
| 21 | `?? ` vs `\|\|`? | `??` falls back only on `null`/`undefined`; `\|\|` on any falsy value, so `""` and `0` trigger it. `port \|\| 3000` when `PORT="0"` is a real bug. [L03] |
| 22 | `interface` vs `type`? | `type` is more capable (unions, tuples, computed). `interface` gives declaration merging, declaration-site conflict errors, and better compiler performance at scale. Rule: `interface` for object shapes, `type` for everything else. [L04] |
| 23 | Why does an object literal error but a variable doesn't? | **Excess property checks on fresh object literals.** Structural typing allows extra properties; a literal written inline can only be for one target, so an unexpected key is almost certainly a typo. Freshness is lost via a variable, spread or `as`. **This catches React prop typos.** [L04] |
| 24 | `x?: T` vs `x: T \| undefined`? | Optional (may be absent) vs required-but-nullable (author must acknowledge it). Use the second when forgetting the field should be an error. [L04] |
| 25 | What does `satisfies` do? | Validates a value against a type **without widening it**. Annotating loses literal types; omitting checks nothing. `satisfies` gives both — the right tool for config objects, route maps and lookup tables. [L04] |

---

## Part 2 — The type system (25)

| # | Question | Strong answer |
|---|---|---|
| 26 | What's contextual typing? | Parameters of a function expression are inferred from the expected type at that position. It's why callbacks rarely need annotations — and why extracting one to an un-annotated `const` produces implicit `any`. Fix: annotate the *variable*'s function type. [L05] |
| 27 | Why can a 1-param function satisfy a 3-param type? | Extra arguments are ignored in JS, so it's safe — and required for `forEach(x => ...)`. The consequence: `["1","2","3"].map(parseInt)` → `[1, NaN, NaN]`, because `map` passes the index as the radix. **Type-correct, entirely wrong.** [L05] |
| 28 | When are overloads right? | Only when the return type depends on which argument shape was passed, in a way a union can't express. Try a union, then a generic, then a conditional return type first. Overloads also **don't accept union arguments**, which is usually the sign you wanted a union parameter. [L05] |
| 29 | Should you annotate return types? | Exported: yes — it's a contract, errors stay local, `any` can't leak, and it's faster to compile. Internal: usually no; inference is accurate and annotations drift. [L05, L18] |
| 30 | Type predicate vs assertion function? | Predicate (`x is T`) returns a boolean to branch on; assertion (`asserts x is T`) throws and narrows for the rest of the scope. **Both are trusted unconditionally**, so a wrong one is as dangerous as `as`. [L05, L07] |
| 31 | How do you type a wrapper preserving the signature? | `function wrap<A extends unknown[], R>(fn: (...a: A) => R): (...a: A) => R`. The pattern for logging, retry, memoize, throttle. [L05] |
| 32 | Structural vs nominal typing? | Compatibility by shape vs declared name. TypeScript had to be structural to describe existing JavaScript. Cost: `UserId` and `PostId` are both `string`. Fix: branding. Exceptions where TS *is* nominal: `private` class members and `enum`. [L06] |
| 33 | Explain covariance and contravariance. | Covariant = same direction (returns, `readonly T[]`, `Promise<T>`). Contravariant = flipped (function parameters). **The reasoning:** `Handler<Animal>` fits where `Handler<Dog>` is expected because it'll only be called with dogs and it handles all animals; the reverse lets a `Cat` reach code reading `.breed`. *"A function accepting more is substitutable where less is expected."* [L06] |
| 34 | Why does pushing `(e: MouseEvent) => void` into `Array<(e: Event) => void>` compile? | `Array.push` is a **method**, and method parameters are **bivariant**. `strictFunctionTypes` only checks function-type *properties* — deliberately, because the standard library and DOM depend on bivariant methods. [L06] |
| 35 | Is `Dog[]` assignable to `Animal[]`? Should it be? | Yes, and it's unsound — you can then `push(new Cat())` and corrupt the original. TypeScript chose convenience because the pattern is idiomatic. `readonly Animal[]` is the sound version: **covariance is only unsafe with writes**. [L06] |
| 36 | How do you get nominal typing? | Branded types — intersect with a phantom property, ideally a `unique symbol`. **Zero runtime cost.** Or a class with a `private` member (real nominal, runtime weight). [L06, L09] |
| 37 | Why is `(x: unknown) => void` assignable to `(x: string) => void`? | Contravariance at its limit: a handler accepting *anything* certainly accepts a string. `unknown` is the top type, so it's the maximally assignable parameter type. [L06] |
| 38 | What's narrowing? | Control-flow analysis: TS simulates your code paths and tracks the narrowest type at each point. Mechanisms: `typeof`, truthiness, equality, `in`, `instanceof`, predicates, assertions, discriminants. [L07] |
| 39 | What's a discriminated union and why does it matter? | A union with a shared literal discriminant, enabling exhaustive narrowing. **It makes illegal states unrepresentable:** `{loading, data?, error?}` permits 8 states of which 3 are meaningful. [L07, L09] |
| 40 | Why doesn't narrowing survive into my callback? | The callback runs later and a reassignable variable might have changed. Fix: copy to a `const` — **or simply don't reassign**, because TS *does* preserve narrowing into closures for `const`s and never-reassigned parameters. [L07] |
| 41 | What's wrong with `if (!count)` for `number \| undefined`? | `0` is falsy, so a valid zero takes the "missing" branch. Use `count === undefined`. [L07] |
| 42 | Why does `filter(x => x !== null)` keep `null` in the type? | `filter`'s signature returns `T[]`; it can't know your callback narrows. Fix with a predicate: `filter((x): x is string => x !== null)` or a reusable `isNotNullish`. TS 5.5 infers predicates for simple cases. [L07] |
| 43 | When is `!` acceptable? | After an invariant the compiler can't see — a `Map.get` right after a `set`, a known DOM node in a test. Never to silence an error you don't understand, and always with a comment naming the invariant. [L07] |
| 44 | Why doesn't `typeof x === "string"` narrow generic `T` to `string`? | It narrows to `T & string`, because `T` could be a subtype with extra structure — so you can use string methods but can't assign back to `T`. Usually a sign you didn't need a generic. [L07, L08] |
| 45 | When do you use a generic? | To preserve a **relationship** between types. **The test:** if a type parameter appears only once in the signature, it preserves nothing and should be `unknown` or concrete. [L08] |
| 46 | What's wrong with `function parse<T>(s: string): T`? | It's `as` in disguise — the caller invents `T`, nothing validates it, and the compiler now trusts an unverified shape. Return `unknown` and validate. **The most common bad generic in the wild.** [L08] |
| 47 | Why does `f("hello")` infer `string` not `"hello"`? | Unconstrained type parameters widen. `<T extends string>` infers the most specific type satisfying the constraint; `<const T>` (TS 5.0) infers as if `as const` were written. [L08] |
| 48 | Constraint vs default? | `<T extends string>` restricts what `T` can be; `<T = string>` only supplies a value when inference has nothing — it restricts nothing. Commonly confused. [L08] |
| 49 | How do you debug bad inference? | Hover first. Then assert the expected type and read the error (it names the actual). `Prettify<T>` to flatten intersections. Check for an unconstrained parameter (widening) or multiple candidates (union). [L08] |
| 50 | Design a typed event emitter. | An event-map type plus `on<K extends keyof M>(e: K, fn: (p: M[K]) => void)` and `emit<K extends keyof M>(e: K, p: M[K])`. Unknown names and wrong payloads both become compile errors. [L08] |

---

## Part 3 — Type-level & design (25)

| # | Question | Strong answer |
|---|---|---|
| 51 | "Make illegal states unrepresentable" — what does it mean? | Design types so representable values = valid values. **Give the counting argument:** 8 states vs 3 meaningful; the other 5 are constructible bugs, and every consumer writes defensive code the compiler can't verify. [L09] |
| 52 | What's a branded type and what does it cost? | An intersection of a primitive with a phantom property (ideally a `unique symbol`) giving nominal behaviour. **Runtime cost: zero.** Stops ID mix-ups, seconds-vs-millis, and raw-vs-sanitised strings. [L09] |
| 53 | How do you stop someone adding USD to EUR? | Parameterise `Money<C extends Currency>` and make `add<C>(a: Money<C>, b: Money<C>)`. Mismatched currencies fail to unify → compile error. A financial bug testing catches unreliably. [L09] |
| 54 | `Result` or exceptions? | `Result` for **expected** domain failures the caller must handle (the failure becomes part of the signature). Exceptions for programmer errors. **Boundary rule:** `Result` at module edges, exceptions inside — don't thread it through six helpers. [L09, L16] |
| 55 | What's "parse, don't validate"? | A validator returns a boolean and leaves the value untrusted; a **parser returns a narrower type carrying the proof**. Once you hold an `Email`, the check provably happened. [L09, L15] |
| 56 | What's a conditional type? | `A extends B ? X : Y` — an `if` at the type level, where `extends` means "is assignable to". With `infer` it's pattern matching. [L10] |
| 57 | What's distribution? | A conditional over a **naked type parameter** applies per union member and unions the results. `ToArray<string\|number>` = `string[] \| number[]`. Disable with `[T] extends [U]`. It's how `Exclude` works. [L10] |
| 58 | How do you check if a type is `never`? | `[T] extends [never] ? true : false`. The naïve version distributes over an **empty union**, so the body never runs and you get `never`. [L10] |
| 59 | Why does `Omit` break my discriminated union? | `Omit` = `Pick<T, Exclude<keyof T, K>>`, and `keyof (A\|B)` is only the *shared* keys, so variants collapse. Fix: `T extends any ? Omit<T,K> : never`. [L10, L12] |
| 60 | Implement `ReturnType`. | `T extends (...a: any) => infer R ? R : never`. Then explain `infer`: bind whatever occupies that position. [L10] |
| 61 | Explain `UnionToIntersection`. | Distribute the union into functions taking each member, then `infer` the parameter — **contravariant position, so multiple candidates intersect rather than union**. Variance exploited at the type level. [L10] |
| 62 | Why does `any extends string ? A : B` give `A \| B`? | `any` is assignable both ways, so the checker can't decide and returns both branches. It's how a leaked `any` turns computed types into confusing unions. [L10] |
| 63 | What's a mapped type? | `{ [K in keyof T]: ... }` — iterate keys to build a new object type, with `?`/`-?` and `readonly`/`-readonly` modifiers. Every utility type is one. [L11] |
| 64 | How do you remove keys in a mapped type? | Key remapping with `as`, mapping unwanted keys to `never`. That's how `PickByValue` and "omit all methods" work. [L11] |
| 65 | What's a homomorphic mapped type? | One mapping directly over `keyof T`. It preserves `readonly`/`?` and **distributes over arrays, tuples and unions** — which is why `Partial<string[]>` stays an array. Certain `as` clauses break it. [L11] |
| 66 | What are template literal types for? | Type-safe strings: event names, route paths with typed params, CSS scales, i18n keys. Plus `infer` gives type-level string parsing — how typed `get(obj, "a.b.c")` works. Watch the combinatorial explosion. [L11] |
| 67 | Type a component prop that's *either* `href` or `onClick`. | `XOR<A,B>` — each branch intersected with the other's keys as optional `never`. Compile-time instead of a runtime check. [L11, L17] |
| 68 | Implement `Partial`, `Required`, `Readonly`, `Pick`. | The four one-liners, including `-?` and `-readonly`. [L12] |
| 69 | What's wrong with `Omit`? | Three things: it doesn't constrain `K` to `keyof T` (typos are **silent**), it collapses unions, and people use it where `Pick` would be clearer. Keep `StrictOmit` and `DistributiveOmit`. [L12] |
| 70 | `Exclude<"a" \| "b", string>` = ? | `never` — `Exclude` removes members **assignable to** `U`, and every string literal is assignable to `string`. Exact removal needs a type-equality check. [L12] |
| 71 | Implement type equality. | `type Equal<X,Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false`. Two generic signatures are only mutually assignable if their deferred conditionals are identical. [L12] |
| 72 | How do you type an API enum that might gain values? | `LiteralUnion<"a"\|"b", string>` = `"a" \| "b" \| (string & {})`. Known values autocomplete; new ones still parse. `& {}` prevents literal absorption. Pair with an exhaustive switch that has a `default`. [L12] |
| 73 | How do you test types? | `.test-d.ts` with `Equal`/`Expect`, or `expect-type`/`tsd`. Plus **`@ts-expect-error` as a positive assertion that something must fail** — the only way to stop a type design silently loosening. [L12] |
| 74 | When do you stop writing clever types? | When you can't explain it in 60 seconds, when the errors become unreadable, when compile time moves, or when it's application rather than library code. *"A clever type in a feature file has one beneficiary and a permanent cost."* [L10, L12] |
| 75 | When would you reach for `type-fest`? | Deep/recursive transformations and anything with edge cases (`Jsonify`, `Merge`, `PartialDeep`). One-liners you understand, write yourself. *"The line is whether I can enumerate the edge cases."* [L12] |

---

## Part 4 — Real code & ecosystem (25)

| # | Question | Strong answer |
|---|---|---|
| 76 | When do you use a class in TypeScript? | Stateful objects with a lifecycle, error types (`instanceof` needs a runtime construct), resource handles, framework requirements. **The reasoning:** modules are already singletons and imports are already DI, so a stateless service class is a container without contents. [L13] |
| 77 | `private` vs `#private`? | `private` is compile-time only — erased, reachable via `obj["x"]`, **visible in `JSON.stringify`**. `#` is a real runtime private field. Use `#` for secrets. `private` also makes the class nominally typed. [L13] |
| 78 | Why does `instanceof` fail on my custom Error? | Downlevelling to ES5 breaks built-in subclassing. Fix: `Object.setPrototypeOf(this, new.target.prototype)` or target ES2015+. And use `Error.cause` to chain rather than string concatenation. [L13] |
| 79 | Why did `this` become `undefined`? | The method was extracted from its receiver. Fix: an arrow class field (bound at construction — costs a closure per instance and isn't overridable) or wrap at the call site. Why React class components needed `.bind(this)`. [L13] |
| 80 | What happens to a class instance through JSON? | Prototype gone, methods gone, `instanceof` false. **A strong argument for plain objects + types at every boundary** — network, storage, workers, server/client component boundaries. [L13] |
| 81 | Does `implements` do anything? | It's a declaration-site check for a clearer, earlier error. It creates **no type relationship** (structural typing already handles that) and provides **no contextual typing**, so method parameters still need annotations. [L13] |
| 82 | "Cannot find module x" — walk me through it. | The resolution order: relative → `paths` → package `exports.types` → `types`/`typings` → sibling `.d.ts` → `@types/*` → local `declare module`. Then **`tsc --traceResolution`** to see which step failed. Usual causes: no types shipped, or `moduleResolution: "Node10"` can't read a modern `exports` map. [L14] |
| 83 | What makes a file a module vs a script? | Any top-level `import`/`export`. Without one, declarations are **global** — the cause of cross-file duplicate-identifier errors. `export {}` fixes it. [L14] |
| 84 | How do you add a field to `Express.Request`? | `declare global { namespace Express { interface Request { … } } }` plus `export {}`. **Declaration merging** — the concrete reason `interface` still matters. [L14] |
| 85 | What's wrong with barrel files? | Importing one symbol loads every module in the barrel; they create circular-import risk and slow the compiler. Fine at a published package's root; avoid internally. Next.js and Vite both ship optimisations for this. [L14] |
| 86 | What breaks with circular imports? | Types are fine; **values** aren't — in ESM one side sees `undefined` at runtime rather than erroring. Fix by extracting shared code or using `import type`. [L14] |
| 87 | Where is "the boundary"? | HTTP responses, `JSON.parse`, `localStorage`, env vars, URL params, forms, WebSocket messages, third-party callbacks, `catch` variables, and (softly) database rows. [L15] |
| 88 | What's wrong with `JSON.parse(raw) as User`? | `as` disables checking — it's not validation. Nothing ran, and the error surfaces later with a misleading stack trace. [L15] |
| 89 | Why Zod over a hand-written guard? | **One source of truth.** With a guard you maintain an interface *and* a check and they drift; `z.infer` makes drift impossible. Plus transforms, per-field error paths, and cross-field `refine`. [L15] |
| 90 | `parse` vs `safeParse`? | `parse` throws — for failures that mean a bug (env at boot). `safeParse` returns a result — for expected failures (user input, third-party responses). [L15] |
| 91 | `z.input` vs `z.infer`? | With `.transform()`, input (what the wire/form sends) differs from output (what your code holds). It's how "strings on the wire, `Date` and minor units internally" lives in one declaration. [L15] |
| 92 | Should you validate your own API's responses in production? | Nuance: third parties always; your own API in dev/staging at minimum, and usually in production too — the cost is negligible next to the round trip and the diagnostic value on contract drift is high. Sample if payloads are huge. [L15] |
| 93 | Why is `catch (e)` `unknown`? | JavaScript can throw anything — a string, `null`, a plain object. Pre-`strict` it was `any`, so `e.message` was a silent `undefined`. That's how you get `Error: undefined` in logs. [L16] |
| 94 | What's wrong with `items.forEach(async x => await save(x))`? | `forEach` ignores returned promises — nothing is awaited, errors vanish. TS allows it because `() => Promise<void>` is assignable to `() => void`. Use `for...of`, `Promise.all`, or a bounded `mapLimit`. [L16] |
| 95 | What's a floating promise? | A promise nobody awaits or catches. In Node 15+ an unhandled rejection **terminates the process**. Enforce `no-floating-promises`; use `void p.catch(log)` for deliberate fire-and-forget. [L16] |
| 96 | Three sequential awaits that don't depend on each other? | Three round trips instead of one — `Promise.all` them. One of the most common real performance bugs, and invisible in review unless you look for it. [L16] |
| 97 | How do you type a React component? | A plain function with a props type; extend `ComponentPropsWithoutRef<"button">` so consumers get native attributes. On `React.FC`: the implicit-`children` objection was fixed in React 18; the remaining reason to avoid it is generic components. [L17] |
| 98 | How do you type context so consumers never handle `undefined`? | Default to `undefined`, export a hook that **throws** if missing. The throw narrows, so consumers get a non-nullable value and a missing provider fails loudly. [L17] |
| 99 | Why does destructuring a custom hook's return lose types? | An array return infers as `(A\|B)[]`, not a tuple. `as const` fixes it. Object return for 3+ values. [L17] |
| 100 | How would you migrate 200k lines of JS to TS? | Six phases: ratchet in CI → type dependencies → convert leaves first → `checkJs` + JSDoc for the rest → strict flags one at a time (`strictNullChecks` last) → per-directory strict config or error-count baseline. Plus the two rules: **convert-don't-refactor**, and ride along with feature work. [L19] |

---

## Part 5 — Judgement (10)

| # | Question | What a strong answer contains |
|---|---|---|
| 101 | When is `any` acceptable? | Genuinely dynamic metaprogramming, a third-party type you can't fix (contained in **one adapter module**), and migration placeholders. Always with a comment. **Prefer `unknown`** — it's the only one where the compiler keeps helping. |
| 102 | When is `as` acceptable? | Inside a **validating constructor** for a branded type (one audited place), after a check the compiler can't see, and in tests. Never to make an error disappear — that converts a compile error into a runtime one. |
| 103 | How much type complexity is too much? | When you can't explain it in 60 seconds, when errors become unreadable, when compile time moves, or when it's in feature code. Library and application code have genuinely different budgets. |
| 104 | Does TypeScript slow you down? | Up front, sometimes — and it pays back superlinearly with codebase size and team size, because refactoring becomes mechanical. Where it genuinely costs: heavy metaprogramming and badly-typed dependencies. *"If I'm fighting the compiler for an hour, I'm usually modelling the domain wrong."* |
| 105 | What would you change about TypeScript? | Real opinions, well-reasoned: soundness holes (array covariance, method bivariance); `enum` being non-erased and nominal; no first-class nominal types without branding; `Object.keys` returning `string[]` (correct but inconvenient); no runtime validation story in the language. Showing you know *why* each exists matters more than the complaint. |
| 106 | How do you review TypeScript? | `any`/`as`/`!` first; then boundary validation; then whether illegal states are representable; then exported return-type annotations; then complexity. Grade findings — a missing boundary validation outranks a naming nit. [L22] |
| 107 | How do you onboard someone onto a strict codebase? | Document the conventions (when `any`, when `as`, what needs validation), pair on the first boundary they touch, and point them at the domain types first — the branded IDs and discriminated unions teach the model faster than any doc. |
| 108 | Types or tests? | Both, for different failures. Types eliminate whole *categories* (wrong shape, missing case, illegal state); tests verify *behaviour*. Strong types make tests shorter, because you stop testing what the compiler proves. |
| 109 | What's the most valuable TypeScript habit? | **Hover over everything.** The inferred type is the compiler telling you what it concluded. People who become fluent read it constantly; people who stay stuck only read red squiggles. |
| 110 | What does TypeScript *not* fix? | Wrong logic, bad domain models, unvalidated inputs, missing error handling, race conditions, and performance. **It stops you saying the wrong thing; it doesn't stop you meaning the wrong thing.** |

---

## The 15-question self-test (cold, timed)

From [the README](../README.md). Two minutes each, out loud, no notes.

1. What does `tsc` do at runtime?
2. `any` vs `unknown` vs `never` vs `void`, with a use for each
3. Why does pushing `(e: MouseEvent) => void` into `Array<(e: Event) => void>` compile?
4. `interface` vs `type` — three differences
5. Why `const x = 'a'` gives `'a'` but `let` gives `string`
6. What's an excess property check, and why does a variable bypass it?
7. What does `noUncheckedIndexedAccess` change, and why is it off by default?
8. Write a type predicate; then an assertion function
9. How do you make a switch exhaustive?
10. Implement `Omit`; explain why it doesn't distribute
11. What does `infer` do? Implement `ReturnType`
12. What's a branded type and what does it solve?
13. Your API returns `{id: number}` but the type says `string` — when do you find out?
14. `enum` vs literal union vs `as const` object
15. Your build takes 90 seconds — how do you find out why?

**18–20 confident → ready. 14–17 → drill the gaps. Under 14 → re-read the modules; you're recognising, not recalling.**

---

## What's next

Rapid-fire tests recall. The other TypeScript interview format is a **live type challenge** — implement a utility type on a shared screen — which is a performance skill you have to practise.

Next → **[Lesson 21: Type challenges](21-type-challenges.md)**
