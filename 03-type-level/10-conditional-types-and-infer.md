# Lesson 10 — Conditional types & `infer`

> **Why this lesson exists:** conditional types are where TypeScript becomes a (pure, functional, lazily-evaluated) programming language whose values are types. This is how every utility type is built, how libraries achieve their "how does it know?" inference, and — crucially — how you *read* the type gymnastics in other people's code instead of scrolling past it. It's also where distribution over unions surprises everyone exactly once.

**Time:** ~70 minutes · **Prereq:** Module 2

---

## 1. The idea in one sentence

> **`A extends B ? X : Y` is an `if` statement at the type level, `infer` is its variable declaration, and the one rule you must know is that a conditional type over a *naked type parameter* **distributes over unions**.**

---

## 2. The basic form

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">;    // true
type B = IsString<42>;          // false
```

`extends` here means **"is assignable to"** — the same relation from [Lesson 06](../02-type-system/06-structural-typing-and-variance.md), not class inheritance.

```ts
type T1 = "a" extends string ? 1 : 0;                  // 1 — literal is assignable to string
type T2 = string extends "a" ? 1 : 0;                  // 0 — not the other way
type T3 = { a: 1; b: 2 } extends { a: 1 } ? 1 : 0;     // 1 — structural, extra props fine
type T4 = never extends string ? 1 : 0;                // 1 — never is assignable to everything
type T5 = any extends string ? 1 : 0;                  // 1 | 0  ← `any` takes BOTH branches!
```

That last one is a genuine oddity worth knowing: **`any` in the checked position yields a union of both branches**, because `any` is assignable to and from everything. It shows up when a stray `any` leaks into a generic and your computed type becomes a confusing union.

### Practical uses straight away
```ts
type NonNullish<T> = T extends null | undefined ? never : T;
type A = NonNullish<string | null>;          // string   (never vanishes from a union)

type Flatten<T> = T extends readonly (infer U)[] ? U : T;
type B = Flatten<string[]>;                   // string
type C = Flatten<number>;                     // number
```

---

## 3. Distribution — the rule that surprises everyone

**When the checked type is a *naked type parameter* and you pass a union, the conditional applies to each member separately and the results are unioned.**

```ts
type ToArray<T> = T extends any ? T[] : never;

type A = ToArray<string | number>;
// NOT (string | number)[]
// IS  string[] | number[]        ← distributed
```

Step by step: `ToArray<string | number>` → `ToArray<string> | ToArray<number>` → `string[] | number[]`.

This is what makes filtering unions possible, and it's how `Exclude` works:
```ts
type Exclude2<T, U> = T extends U ? never : T;

type Status = "loading" | "success" | "error";
type Settled = Exclude2<Status, "loading">;
// "loading" extends "loading" ? never : "loading"  → never
// "success" extends "loading" ? never : "success"  → "success"
// "error"   extends "loading" ? never : "error"    → "error"
// union: never | "success" | "error"  =  "success" | "error"   ✅
```
**`never` disappears from a union** — that's why `never` is the "delete this member" value.

### Turning distribution OFF

Wrap both sides in a tuple (`[T] extends [U]`):

```ts
type IsUnion<T, U = T> = T extends any ? ([U] extends [T] ? false : true) : never;

// The canonical case — checking for `never`:
type IsNever1<T> = T extends never ? true : false;
type A1 = IsNever1<never>;        // never  ← ❌ distributing over an EMPTY union gives never!

type IsNever2<T> = [T] extends [never] ? true : false;
type A2 = IsNever2<never>;        // true   ✅
```

**`never` is an empty union, so distributing over it produces `never`** — the loop body never runs. `[T] extends [never]` is the idiom for testing it, and it's a classic interview question.

```ts
// Another case where you want distribution off:
type AllStrings<T> = [T] extends [string] ? true : false;
type B1 = AllStrings<"a" | "b">;   // true  — the whole union is strings
type B2 = AllStrings<"a" | 1>;     // false
```

### The practical consequence: `Omit` doesn't distribute

```ts
type Payment =
  | { status: "succeeded"; id: string; capturedAt: string }
  | { status: "failed"; id: string; failureCode: string };

type WithoutId = Omit<Payment, "id">;
// ❌ { status: "succeeded" | "failed" }   ← the union COLLAPSED
```

`Omit<T, K>` is defined as `Pick<T, Exclude<keyof T, K>>`, and `keyof (A | B)` is only the **shared** keys — so the variants merge and you lose the discriminant's correlation.

```ts
// ✅ A distributive version
type DistributiveOmit<T, K extends PropertyKey> =
  T extends any ? Omit<T, K> : never;

type Fixed = DistributiveOmit<Payment, "id">;
// { status: "succeeded"; capturedAt: string } | { status: "failed"; failureCode: string }  ✅
```

**This is a real bug people hit constantly** — usually when writing a React component's props as `Omit<SomeUnionProps, "onChange">` and wondering why narrowing stopped working. Keep `DistributiveOmit` in your utils.

---

## 4. `infer` — pattern matching for types

`infer X` says *"whatever is in this position, call it `X`."* It's destructuring at the type level.

```ts
type ReturnType2<T> = T extends (...args: any[]) => infer R ? R : never;
type A = ReturnType2<() => string>;                 // string

type Parameters2<T> = T extends (...args: infer P) => any ? P : never;
type B = Parameters2<(a: string, b: number) => void>;   // [a: string, b: number]

type ElementOf<T> = T extends readonly (infer E)[] ? E : never;
type C = ElementOf<Payment[]>;                       // Payment

type PromiseValue<T> = T extends Promise<infer V> ? V : T;
type D = PromiseValue<Promise<Payment>>;             // Payment
```

### Recursive `infer` — `Awaited`
```ts
type Awaited2<T> =
  T extends null | undefined ? T
  : T extends object & { then(onfulfilled: infer F): any }
    ? F extends (value: infer V, ...args: any) => any ? Awaited2<V> : never
  : T;

type A = Awaited2<Promise<Promise<string>>>;   // string  — unwraps all the way
```
Conditional types can recurse. TypeScript has a depth limit (~50 for type instantiation, ~1000 for tail-recursive cases since 4.5) that you'll hit only in genuinely extreme code.

### Multiple `infer`s, and inferring to a constraint
```ts
type FirstArg<T> = T extends (first: infer A, ...rest: infer R) => infer Ret
  ? { first: A; rest: R; returns: Ret } : never;

// Constrained infer (TS 4.7+) — infer only if it matches
type FirstString<T> = T extends [infer S extends string, ...unknown[]] ? S : never;
type A2 = FirstString<["a", 1]>;      // "a"
type B2 = FirstString<[1, "a"]>;      // never
```

### Multiple `infer`s with the same name
```ts
// In a COVARIANT position → union
type Cov<T> = T extends { a: infer U; b: infer U } ? U : never;
type A3 = Cov<{ a: string; b: number }>;     // string | number

// In a CONTRAVARIANT position (parameters) → intersection
type Contra<T> = T extends { a: (x: infer U) => void; b: (x: infer U) => void } ? U : never;
type B3 = Contra<{ a: (x: string) => void; b: (x: number) => void }>;   // string & number = never
```
That's variance ([Lesson 06](../02-type-system/06-structural-typing-and-variance.md)) showing up at the type level — and it's the trick behind `UnionToIntersection`:
```ts
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void ? I : never;

type X = UnionToIntersection<{ a: 1 } | { b: 2 }>;   // { a: 1 } & { b: 2 }
```
Read it as: distribute the union into a union of *functions taking* each member, then infer the parameter — which, being contravariant, must be the intersection. **This is the most famous type-level trick in TypeScript**, and being able to explain it is a real flex.

---

## 5. The patterns you'll actually write

### Dependent return types
```ts
// The return type depends on an argument's literal type
function getPayment<T extends boolean>(
  id: PaymentId,
  expand: T,
): Promise<T extends true ? PaymentWithCustomer : Payment>;

// Better: a lookup-based version, which reads more clearly
type ResponseFor<K extends keyof Routes> = Routes[K]["response"];
```
Overloads are often clearer than a conditional return type ([Lesson 05](../02-type-system/05-functions-and-overloads.md)). **Reach for a conditional return type when the variation is over a generic, not over a fixed set of literals.**

### Optional properties from conditions — the API-client pattern
This is the mechanism behind the typed client from Lesson 08:
```ts
type HasBody<R>   = R extends { body: infer B } ? { body: B } : {};
type NeedsKey<R>  = R extends { idempotent: true } ? { idempotencyKey: IdempotencyKey } : {};

type RequestOptions<K extends keyof Routes> = HasBody<Routes[K]> & NeedsKey<Routes[K]>;
// `POST /v1/payments` REQUIRES an idempotency key; `GET /v1/balance` forbids extra options.
```
That's the API's *"idempotency key required on money-moving POSTs"* rule, enforced by the compiler.

### Deep transformations
```ts
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

type DeepReadonly<T> = T extends (infer E)[]
  ? readonly DeepReadonly<E>[]
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```
⚠️ Naïve deep types break on `Date`, `Map`, `Set`, `RegExp` and functions — they're objects, so they get mapped into `{}`-shaped nonsense.
```ts
type Primitive = string | number | boolean | bigint | symbol | null | undefined;
type Builtin = Primitive | Date | RegExp | ((...a: any[]) => any);

type DeepReadonly2<T> =
  T extends Builtin ? T
  : T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly2<K>, DeepReadonly2<V>>
  : T extends Set<infer E> ? ReadonlySet<DeepReadonly2<E>>
  : T extends readonly (infer E)[] ? readonly DeepReadonly2<E>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly2<T[K]> }
  : T;
```
**The built-in exclusion list is the part people forget**, and it's why the naïve version "works" until someone puts a `Date` in the object.

### Extracting from unions
```ts
type ExtractByStatus<T, S> = T extends { status: S } ? T : never;
type Succeeded = ExtractByStatus<Payment, "succeeded">;
// (This is just the built-in `Extract<T, U>` specialised — use `Extract` in real code.)
```

---

## 6. Reading someone else's type gymnastics

A method that works. Given:
```ts
type Paths<T, P extends string = ""> = T extends object
  ? { [K in keyof T & string]: T[K] extends object
        ? Paths<T[K], `${P}${K}.`> | `${P}${K}`
        : `${P}${K}` }[keyof T & string]
  : never;
```

1. **Find the outermost conditional.** `T extends object ? ... : never` — non-objects produce nothing.
2. **Find the mapped type.** `{ [K in keyof T & string]: ... }[keyof T & string]` — this is the **"map then index"** idiom, which produces a *union of the values*. Recognise it; it's everywhere.
3. **Read the recursion.** For an object-valued key, recurse with the prefix extended; otherwise emit the path.
4. **Substitute a tiny example.** `Paths<{ a: { b: string } }>` → `"a" | "a.b"`.

> **The habit that makes this tractable:** paste it into the Playground and instantiate it with a two-property object. **Hovering the result beats reasoning about it**, every time.

### When to *stop*
| Signal | What it means |
|---|---|
| You can't explain the type to a colleague in 60 seconds | It'll be unmaintainable |
| The error messages it produces are unreadable | Your users will hate it |
| Compile time noticeably increased | You've hit the checker's cost curve ([L18](../05-ecosystem/18-tooling-and-performance.md)) |
| It's in application code, not a library | Almost always the wrong place |

**Library code has a different budget from application code.** In a library, you pay the complexity once and thousands of consumers get better inference. In an app, a clever type is a maintenance liability with one beneficiary.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Remember distribution: naked type parameter + union = per-member** | It's the mechanism behind `Exclude`, and the cause of surprising results |
| **`[T] extends [U]` to switch distribution off** | Required for `never` checks and whole-union checks |
| **Use `DistributiveOmit`, not `Omit`, on unions** | Plain `Omit` collapses a discriminated union |
| **Exclude built-ins (`Date`, `Map`, `Set`, functions) from deep types** | Otherwise they get mapped into nonsense |
| **Prefer overloads or a lookup map to conditional return types when the cases are fixed** | Clearer, better errors |
| **Keep type-level complexity in libraries and shared utils, not in feature code** | Complexity is paid once in a library, forever in an app |
| **Instantiate with a tiny example in the Playground to understand any type** | Hovering beats reasoning |
| **Name intermediate types instead of nesting five conditionals** | Readability, and better error messages |
| **Cap recursion; watch for "Type instantiation is excessively deep"** | The checker has real limits |

---

## 8. Interview traps

**Q1. "What's a conditional type?"**
`A extends B ? X : Y` — an `if` at the type level, where `extends` means "is assignable to." Combined with `infer` it's pattern matching, and it's how every utility type is built.

**Q2. "What is distribution and when does it happen?"**
When the checked type is a **naked type parameter** and you instantiate it with a union, the conditional applies per member and the results are unioned. `ToArray<string|number>` is `string[] | number[]`, not `(string|number)[]`. Turn it off with `[T] extends [U]`.

**Q3. "How do you check if a type is `never`?"**
`[T] extends [never] ? true : false`. The naïve `T extends never` distributes over an **empty union**, so the body never runs and the result is `never` rather than `true`. Classic trick question.

**Q4. "Why does `Omit` break my discriminated union?"**
`Omit<T,K>` is `Pick<T, Exclude<keyof T, K>>`, and `keyof (A|B)` is only the shared keys — so the variants collapse into one object with the common properties. Fix with a distributive version: `T extends any ? Omit<T,K> : never`.

**Q5. "Implement `ReturnType`."**
```ts
type ReturnType2<T extends (...a: any) => any> = T extends (...a: any) => infer R ? R : never;
```
Then explain `infer R`: "whatever occupies the return position, bind it to `R`."

**Q6. "Explain `UnionToIntersection`."**
Distribute the union into a union of functions taking each member, then `infer` the parameter type. Because parameter positions are **contravariant**, multiple `infer` candidates in that position produce an **intersection** rather than a union. It's variance exploited at the type level — and worth being able to derive, not just recite.

**Q7. "Why does `any extends string ? A : B` give `A | B`?"**
`any` is assignable both to and from everything, so the checker can't decide the branch and returns the union of both. It's how a leaked `any` turns computed types into confusing unions.

**Q8. "When do you stop writing clever types?"**
When you can't explain it in 60 seconds, when the resulting errors are unreadable, when compile time moves, or when it's in application rather than library code. *"A clever type in a feature file has one beneficiary and a permanent maintenance cost."*

**Q9. "What's the 'map then index' idiom?"**
`{ [K in keyof T]: SomeType }[keyof T]` — build an object type, then index it by all its keys to get a **union of its values**. It's how you filter keys (`{[K in keyof T]: T[K] extends Fn ? K : never}[keyof T]`) and how most "select keys by value type" utilities work.

---

## 9. Build & break

### Build — implement these from scratch, no lookups
```ts
type MyExclude<T, U>       = ?;    // remove members of U from T
type MyExtract<T, U>       = ?;    // keep only members of T assignable to U
type MyNonNullable<T>      = ?;
type MyReturnType<T>       = ?;
type MyParameters<T>       = ?;
type MyAwaited<T>          = ?;    // recursive
type MyInstanceType<T>     = ?;
type MyOmit<T, K>          = ?;    // then make it distributive
type ElementOf<T>          = ?;
type IsNever<T>            = ?;    // the [T] extends [never] trick
type IsAny<T>              = ?;    // hint: 0 extends (1 & T)
type IsUnion<T>            = ?;
type UnionToIntersection<U>= ?;
type LastInUnion<U>        = ?;    // hard — uses UnionToIntersection
```
Check each against the real one in `lib.es5.d.ts` (cmd-click `Exclude` in your editor). **Reading the standard library's own definitions is one of the best TypeScript exercises there is.**

### Build — Ledger's typed request options
Finish the API client from [Lesson 08](../02-type-system/08-generics-properly.md):
```ts
type HasBody<R>   = R extends { body: infer B } ? { body: B } : {};
type HasParams<R> = R extends { params: infer P } ? { params: P } : {};
type HasQuery<R>  = R extends { query: infer Q } ? { query?: Q } : {};
type NeedsKey<R>  = R extends { idempotent: true } ? { idempotencyKey: IdempotencyKey } : {};

type Options<K extends keyof Routes> =
  HasBody<Routes[K]> & HasParams<Routes[K]> & HasQuery<Routes[K]> & NeedsKey<Routes[K]>;
```
Verify with `@ts-expect-error` tests: a money-moving POST without an idempotency key must not compile.

### Break — five experiments
1. **Distribution.** `type T<X> = X extends any ? X[] : never` with `string | number`. Hover. Then wrap in tuples and hover again.
2. **`IsNever`.** Write both versions and instantiate with `never`. Understand why the naïve one returns `never`.
3. **`Omit` on a union.** Omit a shared key from a discriminated union and watch narrowing break. Fix with `DistributiveOmit`.
4. **Deep types vs `Date`.** Run `DeepReadonly<{ created: Date }>` and hover `created`. It'll be a mapped-over object, not a `Date`. Add the `Builtin` exclusion.
5. **Recursion limit.** Write a type that recurses 60 levels and read the "excessively deep" error.

### Explain out loud (90 seconds)
1. What a conditional type is and what `extends` means there.
2. Distribution: when it happens and how to stop it.
3. The `never` check, and why the naïve version fails.
4. What `infer` does, with `ReturnType` as the example.
5. Your rule for when to stop being clever.

---

## What's next

Conditional types compute *from* types. **Mapped types** transform them — iterating over keys to build new object types — and **template literal types** do the same for strings. Together they're the other half of type-level programming, and the half you'll use more often.

Next → **[Lesson 11: Mapped types & template literal types](11-mapped-and-template-literal-types.md)**
