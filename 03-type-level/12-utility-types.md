# Lesson 12 — The utility types, and building your own

> **Why this lesson exists:** the built-in utility types are TypeScript's standard library, and three of them have famous gotchas that produce silently wrong types (`Omit` and `Exclude` especially). This lesson is the complete catalogue with the traps marked, the implementations so you can rebuild any of them from memory — a real interview exercise — and the judgement call about when to stop writing your own and reach for a library.

**Time:** ~65 minutes · **Prereq:** Lessons 10, 11

---

## 1. The idea in one sentence

> **Every built-in utility type is a short mapped or conditional type you could write yourself in one line — and knowing the implementation is what lets you predict the gotchas instead of being surprised by them.**

---

## 2. The complete catalogue

### Object transformers

| Type | Does | Implementation |
|---|---|---|
| `Partial<T>` | All props optional | `{ [K in keyof T]?: T[K] }` |
| `Required<T>` | All props required | `{ [K in keyof T]-?: T[K] }` |
| `Readonly<T>` | All props readonly | `{ readonly [K in keyof T]: T[K] }` |
| `Pick<T, K>` | Keep only keys `K` | `{ [P in K]: T[P] }` |
| `Omit<T, K>` | Remove keys `K` | `Pick<T, Exclude<keyof T, K>>` ⚠️ |
| `Record<K, V>` | Object with keys `K`, values `V` | `{ [P in K]: V }` |

### Union transformers

| Type | Does | Implementation |
|---|---|---|
| `Exclude<T, U>` | Remove members assignable to `U` | `T extends U ? never : T` ⚠️ |
| `Extract<T, U>` | Keep members assignable to `U` | `T extends U ? T : never` |
| `NonNullable<T>` | Remove `null`/`undefined` | `T & {}` (since 4.9) |

### Function & constructor

| Type | Does |
|---|---|
| `Parameters<T>` | Tuple of parameter types |
| `ReturnType<T>` | Return type |
| `ConstructorParameters<T>` | Constructor parameters |
| `InstanceType<T>` | Instance type of a class |
| `ThisParameterType<T>` | The `this` parameter's type |
| `OmitThisParameter<T>` | The function without its `this` parameter |

### Async & strings

| Type | Does |
|---|---|
| `Awaited<T>` | Recursively unwrap `Promise` |
| `Uppercase` / `Lowercase` / `Capitalize` / `Uncapitalize` | Compiler intrinsics for string literals |

### The ones people forget exist
```ts
type NoInfer<T> = ...;    // TS 5.4 — blocks a position from contributing to inference
function assign<T>(obj: T, defaults: NoInfer<T>): T { /* ... */ }
// Now `defaults` can't widen T — a genuinely useful inference control

// Also: ThisType<T> (contextual `this` in object literals, used by Vue-style APIs)
```

---

## 3. The gotchas — the actual value of this lesson

### Gotcha 1 — `Omit` doesn't check the keys exist

```ts
interface Payment { id: string; money: Money }

type A = Omit<Payment, "id">;        // ✅ { money: Money }
type B = Omit<Payment, "idd">;       // ✅ NO ERROR — silently does nothing
type C = Pick<Payment, "idd">;       // ❌ error — Pick DOES check
```

`Omit<T, K>` is declared as `K extends keyof any` (i.e. any string/number/symbol), not `K extends keyof T`. **A typo in an `Omit` is silent**, and you get a type with the property still present.

```ts
// ✅ A strict version — keep this in your utils
type StrictOmit<T, K extends keyof T> = Omit<T, K>;
type D = StrictOmit<Payment, "idd">;   // ❌ now it errors
```

**Why the standard library is lenient:** `Omit` is often used on unions and generics where `keyof T` isn't statically known, and a strict constraint would break those cases. It's a deliberate trade, and knowing it makes you look like you've read the lib.

### Gotcha 2 — `Omit` collapses discriminated unions

Covered in [Lesson 10](10-conditional-types-and-infer.md), and the most damaging of the three:

```ts
type Payment =
  | { status: "succeeded"; id: string; capturedAt: string }
  | { status: "failed"; id: string; failureCode: string };

type Bad = Omit<Payment, "id">;
// { status: "succeeded" | "failed" }   ← the union COLLAPSED, narrowing is dead

type DistributiveOmit<T, K extends PropertyKey> = T extends any ? Omit<T, K> : never;
type Good = DistributiveOmit<Payment, "id">;   // ✅ still a union
```
Same applies to `Pick`, `Partial` and any non-distributive utility over a union — though `Partial` is homomorphic and *does* distribute ([Lesson 11](11-mapped-and-template-literal-types.md)).

### Gotcha 3 — `Exclude` is about assignability, not identity

```ts
type A = Exclude<string | number, string>;     // number         ✅ as expected
type B = Exclude<"a" | "b", string>;            // never          ⚠️ both are strings!
type C = Exclude<string, "a">;                  // string         ⚠️ can't remove a literal from string
type D = Exclude<{ a: 1 } | { a: 1; b: 2 }, { a: 1 }>;  // never  ⚠️ both are assignable
```
`Exclude<T, U>` removes members **assignable to** `U`, not members *equal to* `U`. For a subtype relation like `"a" extends string`, everything gets removed.

For exact-match removal you need a type-equality check:
```ts
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;
type ExcludeExact<T, U> = T extends any ? (Equal<T, U> extends true ? never : T) : never;

type E = ExcludeExact<"a" | "b", string>;      // "a" | "b"  ✅
```

### Gotcha 4 — `Partial` makes checking harder, not easier
```ts
function update(patch: Partial<Payment>) {
  patch.money.amountMinor;      // ❌ possibly undefined
}
```
`Partial<T>` is right for *"any subset may be provided"* but wrong for *"exactly these fields, each optional with meaning"*. For PATCH semantics you usually want an explicit type ([Lesson 04](../01-foundations/04-objects-and-interfaces.md)):
```ts
type CustomerPatch = {
  name?: string;             // absent = unchanged
  phone?: string | null;     // absent = unchanged, null = CLEAR
};
```
`Partial<Customer>` can't express the `null`-means-clear distinction, which is the API semantics you actually need.

### Gotcha 5 — `Record<string, T>` lies about presence
```ts
const m: Record<string, number> = {};
m.missing.toFixed(2);          // type says number. Runtime: 💥
```
Fixed by `noUncheckedIndexedAccess` ([Lesson 02](../01-foundations/02-tsconfig-deeply.md)), or by using a `Map`, whose `.get` correctly returns `V | undefined`.

### Gotcha 6 — `ReturnType` on an overloaded function picks the last overload
```ts
declare function f(x: string): string;
declare function f(x: number): number;
type R = ReturnType<typeof f>;     // number  — only the LAST overload is seen
```
`infer` on an overloaded function type resolves to the final signature. There's no clean general fix; if you need all of them, don't use overloads ([Lesson 05](../02-type-system/05-functions-and-overloads.md)).

---

## 4. The utilities worth adding to every project

```ts
// ─── Readability ────────────────────────────────────────────
export type Prettify<T> = { [K in keyof T]: T[K] } & {};

// ─── Safe versions of the built-ins ─────────────────────────
export type StrictOmit<T, K extends keyof T> = Omit<T, K>;
export type DistributiveOmit<T, K extends PropertyKey> = T extends any ? Omit<T, K> : never;
export type StrictExtract<T, U extends T> = Extract<T, U>;

// ─── Optionality control ────────────────────────────────────
export type SetOptional<T, K extends keyof T> = Prettify<Omit<T, K> & Partial<Pick<T, K>>>;
export type SetRequired<T, K extends keyof T> = Prettify<Omit<T, K> & Required<Pick<T, K>>>;
export type SetNullable<T, K extends keyof T> = Prettify<Omit<T, K> & { [P in K]: T[P] | null }>;

// ─── Key selection ──────────────────────────────────────────
export type PickByValue<T, V> = { [K in keyof T as T[K] extends V ? K : never]: T[K] };
export type OmitByValue<T, V> = { [K in keyof T as T[K] extends V ? never : K]: T[K] };
export type RequiredKeys<T> = { [K in keyof T]-?: {} extends Pick<T, K> ? never : K }[keyof T];
export type OptionalKeys<T> = { [K in keyof T]-?: {} extends Pick<T, K> ? K : never }[keyof T];

// ─── Deep variants (with built-ins excluded — Lesson 11) ────
type Builtin = string | number | boolean | bigint | symbol | null | undefined
             | Date | RegExp | ((...a: any[]) => any);

export type DeepPartial<T> =
  T extends Builtin ? T
  : T extends readonly (infer E)[] ? readonly DeepPartial<E>[]
  : T extends object ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

export type DeepReadonly<T> =
  T extends Builtin ? T
  : T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends Set<infer E> ? ReadonlySet<DeepReadonly<E>>
  : T extends readonly (infer E)[] ? readonly DeepReadonly<E>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

// ─── Constraints on objects ─────────────────────────────────
export type RequireAtLeastOne<T, K extends keyof T = keyof T> =
  Omit<T, K> & { [P in K]-?: Required<Pick<T, P>> & Partial<Omit<T, P>> }[K];

export type XOR<A, B> =
  | (A & { [K in Exclude<keyof B, keyof A>]?: never })
  | (B & { [K in Exclude<keyof A, keyof B>]?: never });

// ─── Type-level testing ─────────────────────────────────────
export type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;
export type Expect<T extends true> = T;
export type NotEqual<X, Y> = Equal<X, Y> extends true ? false : true;
```

> **Put these in `src/types/utils.ts` on day one of every project.** `Prettify`, `DistributiveOmit`, `SetOptional` and `Equal`/`Expect` alone will earn their keep within a week.

---

## 5. When to use a library instead

**[`type-fest`](https://github.com/sindresorhus/type-fest)** is the standard collection — well-tested, well-documented, and it handles edge cases yours won't.

```ts
import type { SetOptional, Simplify, Jsonify, Merge, PartialDeep, LiteralUnion } from "type-fest";
```

Notable ones worth knowing by name:
| Type | Does |
|---|---|
| `Simplify<T>` | Their `Prettify` |
| `Jsonify<T>` | What `T` becomes after `JSON.stringify`/`parse` — `Date` → `string`, methods dropped |
| `Merge<A, B>` | Shallow merge where `B` wins (unlike `&`, which produces `never` on conflicts) |
| `LiteralUnion<'a' \| 'b', string>` | Known literals with autocomplete, but any string accepted — **the open-enum type** |
| `PartialDeep`, `ReadonlyDeep` | Correct deep variants with the built-in exclusions |
| `Opaque` / `Tagged` | Their branded types |

**`LiteralUnion` is the one to remember**, because it's the direct answer to API Lesson 11's open-enum problem:
```ts
type Status = LiteralUnion<"succeeded" | "failed", string>;
// Autocomplete suggests the known values; a new server-side status still parses.
// Implemented as: T | (string & {})  ← the `& {}` trick prevents literal absorption
```
That `(string & {})` trick is worth knowing on its own: a bare `"a" | string` collapses to `string` and you lose autocomplete, but `"a" | (string & {})` preserves both.

**The decision:**

| Situation | Do |
|---|---|
| A one-liner you fully understand | Write it |
| Deep/recursive transformations | Use `type-fest` — edge cases are brutal |
| An app with 3 custom utilities | Write them |
| A library exporting types to others | Use `type-fest`, and mark it a peer/dev dependency |
| You've written the same helper twice | Move it to `src/types/utils.ts` |

---

## 6. Testing your types

Type-level code needs tests exactly like runtime code — and it's genuinely satisfying to write.

```ts
// src/types/utils.test-d.ts   (a .test-d.ts file that is type-checked, never run)
import type { Equal, Expect } from "./utils";

type cases = [
  Expect<Equal<Prettify<{ a: 1 } & { b: 2 }>, { a: 1; b: 2 }>>,
  Expect<Equal<SetOptional<{ a: 1; b: 2 }, "b">, { a: 1; b?: 2 }>>,
  Expect<Equal<DistributiveOmit<{ k: "a"; x: 1 } | { k: "b"; y: 2 }, "x">,
               { k: "a" } | { k: "b"; y: 2 }>>,
  Expect<Equal<RequiredKeys<{ a: 1; b?: 2 }>, "a">>,
  Expect<Equal<DeepReadonly<{ d: Date }>, { readonly d: Date }>>,   // Date NOT mangled
];

// And the negative cases — @ts-expect-error fails the build if the line STOPS erroring
// @ts-expect-error — StrictOmit rejects a key that doesn't exist
type _bad = StrictOmit<{ a: 1 }, "nope">;
```

Run them with `tsc --noEmit`, or use **`expect-type`** / **`tsd`** for a nicer API and proper test-runner integration:
```ts
import { expectTypeOf } from "expect-type";
expectTypeOf<Prettify<{ a: 1 } & { b: 2 }>>().toEqualTypeOf<{ a: 1; b: 2 }>();
expectTypeOf(api).toBeCallableWith("GET /v1/payments", { query: { limit: 20 } });
```

**The `@ts-expect-error` pattern is the one to internalise:** it's a *positive* assertion that something must fail. If your type stops being strict, the build breaks. That makes it a real regression test for type design, and it's what you used in [Lesson 09](09-illegal-states-unrepresentable.md).

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Use `StrictOmit`, not `Omit`** | A typo'd key in `Omit` is silent |
| **Use `DistributiveOmit` on unions** | Plain `Omit` collapses discriminated unions and kills narrowing |
| **Know that `Exclude` uses assignability, not equality** | `Exclude<"a"\|"b", string>` is `never` |
| **Don't use `Partial<T>` for PATCH types** | It can't express `null` = clear |
| **Wrap public types in `Prettify`** | Readable hovers and error messages for consumers |
| **Deep utilities must exclude `Date`/`Map`/`Set`/functions** | Or use `type-fest` |
| **`LiteralUnion` for open enums from an API** | Autocomplete for known values, tolerance for new ones |
| **Keep `src/types/utils.ts` with the §4 set** | Every project needs them |
| **Write `.test-d.ts` files for any non-trivial type** | Type-level code regresses silently otherwise |
| **`@ts-expect-error` as a positive test that something must fail** | The only way to test strictness |
| **Reach for `type-fest` before writing a deep/recursive utility** | The edge cases will beat you |

---

## 8. Interview traps

**Q1. "Implement `Omit` without using `Omit`."**
```ts
type MyOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
// or directly:
type MyOmit2<T, K extends PropertyKey> = { [P in keyof T as P extends K ? never : P]: T[P] };
```
Then volunteer the gotcha: the real `Omit` doesn't constrain `K` to `keyof T`, so typos are silent — and it collapses unions.

**Q2. "Why does `Omit` break my discriminated union?"**
`Omit<T,K>` is `Pick<T, Exclude<keyof T, K>>`, and `keyof (A|B)` yields only the *shared* keys, so the variants merge into one object with the common properties. Fix: `T extends any ? Omit<T,K> : never`.

**Q3. "`Exclude<'a' | 'b', string>` — what do you get?"**
`never`. `Exclude` removes members **assignable to** `U`, and every string literal is assignable to `string`. For exact removal you need a type-equality check.

**Q4. "Implement type equality."**
```ts
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;
```
It works because two generic signatures are only mutually assignable if their deferred conditional types are identical — the only reliable equality test in TypeScript, and one you should recognise on sight.

**Q5. "How would you make `Partial` deep?"**
Recursive mapped type — **and immediately name the trap**: you must exclude `Date`, `Map`, `Set`, `RegExp` and functions, or they get mapped into `{}`-shaped nonsense. Then say you'd probably use `type-fest`'s `PartialDeep` rather than maintain it.

**Q6. "How do you type an enum from an API that might gain new values?"**
`LiteralUnion<"a" | "b", string>`, implemented as `"a" | "b" | (string & {})`. Known values autocomplete; unknown values still parse. The `& {}` prevents the literals being absorbed into `string`. **This is the client-side answer to open enums** ([API L11](../../API/02-rest-design/11-versioning-and-evolution.md)) — pair it with an exhaustive switch that has a `default`.

**Q7. "How do you test types?"**
`.test-d.ts` files with `Equal`/`Expect`, or `expect-type`/`tsd`. Plus `@ts-expect-error` as a *positive* assertion that something must fail to compile — which is the only way to prevent a type design silently loosening.

**Q8. "When do you reach for `type-fest`?"**
Deep/recursive transformations, anything with edge cases (`Jsonify`, `Merge`, `PartialDeep`), and library code. One-liners you understand, write yourself. *"The line is whether I can confidently enumerate the edge cases."*

---

## 9. Build & break

### Build — `src/types/utils.ts` plus its tests
Write every utility in §4 and a `.test-d.ts` covering each, including negative cases with `@ts-expect-error`. This file travels with you to every future project.

### Build — the "rebuild the standard library" drill
Close all references. Implement, in order:
```
Partial, Required, Readonly, Pick, Record, Exclude, Extract, NonNullable,
Omit, Parameters, ReturnType, ConstructorParameters, InstanceType, Awaited
```
Then open `lib.es5.d.ts` (cmd-click any of them in your editor) and diff yours against the real ones. **Reading the standard library's own definitions is one of the highest-value hours in this track** — they're short, and every one teaches a technique.

### Break — six experiments
```ts
// 1. Silent Omit typo
type A = Omit<{ id: string; x: number }, "idd">;    // hover — id is still there

// 2. Union collapse
type Payment = { k: "a"; x: 1 } | { k: "b"; y: 2 };
type B = Omit<Payment, "x">;                         // hover — union gone

// 3. Exclude and assignability
type C = Exclude<"a" | "b", string>;                 // never

// 4. Partial and nested access
function f(p: Partial<{ money: { amountMinor: number } }>) { p.money.amountMinor; }  // ❌

// 5. Deep utility vs Date
type D = DeepPartialNaive<{ created: Date }>;        // hover `created` — mangled

// 6. Literal absorption
type E = "a" | "b" | string;                         // hover — just `string`, autocomplete gone
type F = "a" | "b" | (string & {});                  // hover — preserved
```
For each, write one sentence explaining the mechanism. If you can do all six, Module 3 has landed.

### Explain out loud (90 seconds)
1. Three built-in utilities and their implementations.
2. The three `Omit`/`Exclude` gotchas.
3. How you'd test a type.
4. What `LiteralUnion` solves and how it's implemented.
5. Where the line is between writing your own and using `type-fest`.

---

## Module 3 complete — checkpoint

Cold, no notes:

- [ ] The counting argument (representable vs meaningful states)
- [ ] Branded types: what, why, and the zero runtime cost
- [ ] How `Money<C>` prevents cross-currency arithmetic
- [ ] When `Result` beats exceptions — and when it doesn't
- [ ] "Parse, don't validate" in one sentence
- [ ] What a conditional type is; what `extends` means there
- [ ] Distribution: when it happens, how to disable it
- [ ] The `[T] extends [never]` trick and why the naïve version fails
- [ ] `infer`, with `ReturnType` implemented from memory
- [ ] `UnionToIntersection` and the variance trick behind it
- [ ] Mapped types, `-?` and `-readonly`
- [ ] Key remapping with `as`, and removal via `never`
- [ ] Homomorphism and why `Partial<string[]>` still works
- [ ] Template literal types plus `infer` for path parsing
- [ ] The three `Omit`/`Exclude` gotchas
- [ ] `Equal`/`Expect` and `@ts-expect-error` for testing types

---

## What's next

You can compute types. Module 4 puts it to work in real code — starting with classes (where they earn their place and where they don't), then the module system, and then **the single most important practical lesson in the track: the boundary**, where external data becomes typed data.

Next → **[Lesson 13: Classes, `this`, and OOP in TypeScript](../04-real-code/13-classes-and-oop.md)**
