# Lesson 21 — Type challenges

> **Why this lesson exists:** *"implement `Omit` on this shared screen"* is a real interview format at companies that write a lot of TypeScript. It's a performance skill — knowing the techniques isn't the same as producing one under observation in five minutes. This lesson gives you a **solving method**, the eight techniques that cover ~90% of challenges, and a graded set to practise on.

**Time:** ~90 minutes, then practise · **Prereq:** Module 3

---

## 1. The method

Say this out loud when you're handed a challenge. It buys thinking time *and* demonstrates process.

```
1. RESTATE     "So I need a type that takes T and K and produces…"
2. EXAMPLE     Write the smallest concrete input and expected output
3. RECOGNISE   Which of the 8 techniques does this need? (§2)
4. SKELETON    Write the shape, even if the middle is wrong
5. INSTANTIATE Substitute the example mentally, out loud
6. EDGE CASES  never, unions, optional props, readonly, empty
7. VERIFY      Expect<Equal<...>> tests
```

**Step 2 is the one candidates skip and shouldn't.** Writing `MyPick<{a:1,b:2}, "a">` → `{a:1}` on screen before you write any type takes ten seconds, prevents solving the wrong problem, and gives you something to check against.

**Step 5 is what makes you look fluent.** *"So `T` is `{a:1,b:2}`, `keyof T` is `'a'|'b'`, this distributes so it checks `'a' extends 'a'` → keep, `'b' extends 'a'` → never…"* Narrating the substitution is exactly how experienced people actually debug types.

---

## 2. The eight techniques (≈90% coverage)

| # | Technique | Signature move | Use when |
|---|---|---|---|
| 1 | **Mapped type** | `{ [K in keyof T]: ... }` | Transform an object's properties |
| 2 | **Key remapping** | `{ [K in keyof T as Cond ? K : never]: ... }` | Filter or rename keys |
| 3 | **Conditional + `infer`** | `T extends F<infer U> ? U : never` | Extract something from a type |
| 4 | **Distribution** | naked `T extends U ? X : Y` | Filter or transform union members |
| 5 | **Map-then-index** | `{ [K in keyof T]: X }[keyof T]` | Produce a **union** from an object |
| 6 | **Recursion** | `T extends [infer H, ...infer R] ? ... Rec<R>` | Tuples, strings, deep objects |
| 7 | **Template literal + `infer`** | `` T extends `${infer A}.${infer B}` `` | Parse strings |
| 8 | **Variance trick** | `(U extends any ? (x: U) => void : never) extends (x: infer I) => void` | Union → intersection |

**If you recognise which of these eight a challenge needs, you're most of the way there.** Practise the recognition, not just the syntax.

### The three reflexes to have ready
```ts
// Disable distribution
type X<T> = [T] extends [U] ? A : B;

// Force literal inference
type Y<T extends string> = ...;      // or <const T>

// Tuple destructuring
type Head<T> = T extends [infer H, ...unknown[]] ? H : never;
type Tail<T> = T extends [unknown, ...infer R] ? R : never;
```

---

## 3. Worked example — think out loud

> **"Implement `DeepReadonly<T>`."**

**1. Restate:** *"A type that makes every property readonly, recursively, including nested objects and arrays."*

**2. Example:**
```ts
type In  = { a: string; b: { c: number; d: string[] } };
type Out = { readonly a: string; readonly b: { readonly c: number; readonly d: readonly string[] } };
```

**3. Recognise:** mapped type (1) + recursion (6) + a conditional to decide when to stop (3).

**4. Skeleton:**
```ts
type DeepReadonly<T> = { readonly [K in keyof T]: DeepReadonly<T[K]> };
```

**5. Instantiate:** *"`T[K]` for `a` is `string`. That recurses into `DeepReadonly<string>`, which maps over `keyof string`… that gives me the string's methods as readonly properties. **That's wrong** — I need a base case."*

**6. Edge cases — narrate them, this is where the marks are:**
```ts
type DeepReadonly<T> =
  T extends Primitive ? T                                    // ← base case
  : T extends readonly (infer E)[] ? readonly DeepReadonly<E>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

type Primitive = string | number | boolean | bigint | symbol | null | undefined;
```
*"And functions are objects, so `DeepReadonly<() => void>` would map over the function's properties and destroy it. Same for `Date`, `Map` and `Set`. So the base case needs to include them:"*
```ts
type Builtin = Primitive | Date | RegExp | ((...a: any[]) => any);

type DeepReadonly<T> =
  T extends Builtin ? T
  : T extends Map<infer K, infer V> ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends Set<infer E> ? ReadonlySet<DeepReadonly<E>>
  : T extends readonly (infer E)[] ? readonly DeepReadonly<E>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

**7. Verify:**
```ts
type _1 = Expect<Equal<DeepReadonly<{ a: { b: 1 } }>, { readonly a: { readonly b: 1 } }>>;
type _2 = Expect<Equal<DeepReadonly<Date>, Date>>;                 // NOT mangled
type _3 = Expect<Equal<DeepReadonly<() => void>, () => void>>;
```

> **Volunteering the `Date`/function edge cases unprompted is what separates a good answer from a correct one.** Most candidates produce the naïve one-liner and stop; naming what breaks it shows you've used this in anger.

---

## 4. The challenges

Do these in order. **Write the `Expect<Equal<>>` tests first** — it forces step 2 of the method.

```ts
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;
type Expect<T extends true> = T;
```

### Tier 1 — Warm-up (should take <2 min each)
```ts
type MyPick<T, K extends keyof T>       // { [P in K]: T[P] }
type MyReadonly<T>
type MyPartial<T>
type MyRequired<T>
type MyExclude<T, U>
type MyExtract<T, U>
type MyNonNullable<T>
type MyReturnType<T>
type MyParameters<T>
type First<T extends readonly unknown[]>
type Last<T extends readonly unknown[]>
type Length<T extends readonly unknown[]>
type Concat<A extends readonly unknown[], B extends readonly unknown[]>
type Includes<T extends readonly unknown[], U>
type TupleToUnion<T extends readonly unknown[]>
```

### Tier 2 — Core (5–10 min each)
```ts
type MyOmit<T, K extends keyof T>
type DistributiveOmit<T, K extends PropertyKey>
type MyAwaited<T>                     // recursive
type DeepReadonly<T>                  // with the Builtin base case
type DeepPartial<T>
type Mutable<T>                       // remove readonly
type SetOptional<T, K extends keyof T>
type SetRequired<T, K extends keyof T>
type RequiredKeys<T>                  // map-then-index + {} extends Pick<T,K>
type OptionalKeys<T>
type PickByValue<T, V>                // key remapping
type OmitByValue<T, V>
type Getters<T>                       // name → getName(): T  (template literal + Capitalize)
type Merge<A, B>                      // B wins on conflicts
type Flatten<T>                       // one level of array nesting
type IsNever<T>                       // [T] extends [never]
type IsAny<T>                         // hint: 0 extends (1 & T)
type IsUnion<T>
type Reverse<T extends readonly unknown[]>
```

### Tier 3 — Hard (15+ min; these are the differentiators)
```ts
type UnionToIntersection<U>           // the variance trick
type LastInUnion<U>                   // uses UnionToIntersection
type UnionToTuple<U>                  // uses LastInUnion, recursively
type Paths<T>                         // "a" | "a.b" | "a.b.c"
type PathValue<T, P extends string>   // resolve a path back to its type
type Split<S extends string, D extends string>
type Join<T extends readonly string[], D extends string>
type Trim<S extends string>
type Replace<S extends string, From extends string, To extends string>
type ReplaceAll<S extends string, From extends string, To extends string>
type CamelCase<S extends string>      // "foo_bar" → "fooBar"
type SnakeCase<S extends string>      // "fooBar" → "foo_bar"
type DeepCamelCase<T>                 // apply it through an object — a REAL use case
type Chunk<T extends readonly unknown[], N extends number>
type Permutation<T>
type Curry<F>                         // (a,b,c)=>r  →  (a)=>(b)=>(c)=>r
type ParseQueryString<S extends string>   // "a=1&b=2" → { a: "1"; b: "2" }
```

### Tier 4 — Practical (these show up in real code)
```ts
// The ones actually worth having built
type RouteParams<T extends string>        // "/users/:id/posts/:pid" → { id: string; pid: string }
type EventMap<T>                           // typed emitter
type XOR<A, B>
type RequireAtLeastOne<T>
type Serialized<T>                         // Date → string, Money → wire shape
type CreateInput<T>                        // Omit server-owned fields
type Column<T>                             // the per-column-typed table (React L17)
type ApiRoutes                             // the full typed client (TS L08/L10)
```
**Tier 4 is the tier that matters most for actual work** — and in an interview, "I built this for a real reason" beats "I solved this puzzle."

---

## 5. Solutions to the ones people get stuck on

### `RequiredKeys` — the `{} extends Pick<T,K>` trick
```ts
type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K
}[keyof T];
```
*Why it works:* `Pick<T, "b">` for an optional `b` is `{b?: X}`, and `{}` **is** assignable to that (all properties optional). For a required `b` it's `{b: X}`, which `{}` is not assignable to. The `-?` is essential — without it, the mapped type's own optionality interferes.

### `IsAny` — `0 extends (1 & T)`
```ts
type IsAny<T> = 0 extends (1 & T) ? true : false;
```
*Why:* `1 & T` is normally `never`-ish or `1`, and `0 extends 1` is false. But `1 & any` is `any`, and `0 extends any` is **true**. `any` is the only type that makes it true. A pure trick, and a classic.

### `UnionToIntersection` — the variance trick
```ts
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void ? I : never;
```
*Why:* distribute `U` into a union of functions taking each member. Then `infer` the parameter — and **parameter positions are contravariant**, so multiple `infer` candidates in that position produce an **intersection**, not a union ([Lesson 10](../03-type-level/10-conditional-types-and-infer.md)).

### `CamelCase` — template literal recursion
```ts
type CamelCase<S extends string> =
  S extends `${infer H}_${infer T}` ? `${H}${CamelCase<Capitalize<T>>}` : S;

type A = CamelCase<"amount_minor_value">;   // "amountMinorValue"
```
And the genuinely useful application — converting a snake_case API response type to camelCase:
```ts
type DeepCamelCase<T> =
  T extends readonly (infer E)[] ? readonly DeepCamelCase<E>[]
  : T extends object
    ? { [K in keyof T as K extends string ? CamelCase<K> : K]: DeepCamelCase<T[K]> }
    : T;

type Wire = { amount_minor: number; created_at: string; payment_method: { last_4: string } };
type App = DeepCamelCase<Wire>;
// { amountMinor: number; createdAt: string; paymentMethod: { last4: string } }
```
**This is the one to show in an interview**, because it's a real problem (snake_case APIs, camelCase JS) solved with three techniques at once.

### `RouteParams` — the practical one
```ts
type PathParams<T extends string> =
  T extends `${string}:${infer P}/${infer Rest}` ? P | PathParams<Rest>
  : T extends `${string}:${infer P}` ? P
  : never;

type RouteParams<T extends string> =
  [PathParams<T>] extends [never] ? {} : { [K in PathParams<T>]: string };

type A = RouteParams<"/v1/merchants/:mid/payments/:pid">;   // { mid: string; pid: string }
type B = RouteParams<"/v1/balance">;                          // {}
```
Note the `[never]` guard — without it, distribution over an empty union gives `never` rather than `{}`.

---

## 6. Interview delivery

### What to say while you work
- *"Let me write the expected input and output first."*
- *"This needs a mapped type with key remapping, because I'm filtering keys."*
- *"Substituting `{a:1,b:2}`: `keyof T` is `'a'|'b'`, this distributes, so…"*
- *"The edge case here is `Date` — it's an object, so a naïve deep map would destroy it."*
- *"Let me add a test rather than eyeball it."*

### What losing candidates do
| Mistake | Instead |
|---|---|
| Silence while thinking | Narrate — it's most of what's being assessed |
| Diving into syntax before an example | Write input → output first |
| Ignoring edge cases | Name `never`, unions, optionality, `Date`/functions **unprompted** |
| Hardcoding to the example | Instantiate with a *second* example |
| Giving up on the recursion | Write the base case first, then the recursive step |
| Not testing | `Expect<Equal<>>` costs ten seconds |

### If you're stuck
Say so productively: *"I know this needs distribution and recursion — let me write the non-recursive case first and build up."* **Working from a partial solution out loud is a positive signal.** Silence and a blank screen is the only real failure.

### The honest framing — worth having ready
> *"I use maybe a third of this in production code — mapped types, key remapping, conditional types with `infer`. The tuple and string-manipulation ones are mostly puzzle territory; I'd reach for them in a library where the inference payoff is amortised over many consumers, and I'd avoid them in feature code where they cost readability for one beneficiary."*

That answer proves both capability *and* judgement, which is strictly better than proving only capability.

---

## 7. Practice protocol

**Week 1 — Tier 1, until each takes under 2 minutes.** These are muscle memory; you shouldn't be thinking about them.

**Week 2 — Tier 2, timed at 10 minutes each.** Write tests first. Note which of the eight techniques each one needed — **that mapping is what you're actually learning.**

**Week 3 — Tier 3, one per day, narrated out loud.** Record yourself once. Listening back is uncomfortable and extremely effective — you'll hear the silences.

**Week 4 — Tier 4 built into Ledger.** These are the ones you keep.

**Then:** [type-challenges](https://github.com/type-challenges/type-challenges) — ~150 graded problems with an in-editor workflow. Do the easy and medium sets; the hard/extreme ones are recreational rather than professional.

### The self-check after each
```
[ ] Did I write input → output before coding?
[ ] Did I name which technique it needed?
[ ] Did I narrate the substitution out loud?
[ ] Did I raise edge cases unprompted?
[ ] Did I write tests?
[ ] Could I explain it to a colleague in 60 seconds?
```

---

## What's next

You can produce types on demand. The final lesson is the other side of the skill: **reviewing** TypeScript — what to look for, in what order, and how to give feedback that gets acted on.

Next → **[Lesson 22: Code review & design judgement](22-review-and-judgement.md)**
