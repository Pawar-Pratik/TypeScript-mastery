# Lesson 11 — Mapped types & template literal types

> **Why this lesson exists:** mapped types are how you transform an object type — `Partial`, `Readonly`, `Pick` and every "make all these optional/nullable/getter-ified" utility is one. Template literal types let types manipulate *strings*, which sounds like a party trick and is actually how you get type-safe routes, event names, CSS units and i18n keys. Together they're the half of type-level programming you'll use weekly.

**Time:** ~70 minutes · **Prereq:** Lesson 10

---

## 1. The idea in one sentence

> **A mapped type is a `for...of` loop over keys that produces a new object type; a template literal type is string interpolation whose inputs are types — and key remapping (`as`) is where the two combine into something genuinely powerful.**

---

## 2. Mapped types: the basic form

```ts
type Mapped<T> = { [K in keyof T]: T[K] };      // identity — copies T
```

Read it as: *for each `K` in the keys of `T`, produce a property `K` whose type is `T[K]`.*

```ts
type MyPartial<T>  = { [K in keyof T]?: T[K] };
type MyRequired<T> = { [K in keyof T]-?: T[K] };        // -? REMOVES optionality
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type Mutable<T>    = { -readonly [K in keyof T]: T[K] };  // -readonly removes it
type Nullable<T>   = { [K in keyof T]: T[K] | null };
type Stringify<T>  = { [K in keyof T]: string };
```

**The modifiers are the mechanism:** `?`/`-?` add and remove optionality, `readonly`/`-readonly` add and remove readonly. `+` is allowed but implicit (`+?` ≡ `?`).

```ts
interface Payment { id: string; money: Money; capturedAt?: string }

type A = MyPartial<Payment>;    // { id?: string; money?: Money; capturedAt?: string }
type B = MyRequired<Payment>;   // { id: string; money: Money; capturedAt: string }
type C = Mutable<Readonly<Payment>>;  // back to mutable
```

### Mapping over a union of keys, not an object
```ts
type Flags<K extends string> = { [P in K]: boolean };
type F = Flags<"canRefund" | "canCapture">;   // { canRefund: boolean; canCapture: boolean }

// This is exactly how Record works:
type MyRecord<K extends PropertyKey, V> = { [P in K]: V };
```

### Homomorphic mapped types — the subtlety worth knowing
A mapped type of the form `{ [K in keyof T]: ... }` (mapping over `keyof T` directly) is **homomorphic**: it preserves `readonly` and `?` modifiers from `T`, and — importantly — it **distributes over arrays, tuples and unions**.

```ts
type Wrapped<T> = { [K in keyof T]: T[K] };

type A = Wrapped<string[]>;        // string[]      ← stays an array, doesn't become {0:..., length:...}
type B = Wrapped<[string, number]>;// [string, number]  ← stays a tuple
type C = Wrapped<Payment | Refund>;// Wrapped<Payment> | Wrapped<Refund>  ← distributes

type NotHomomorphic<T> = { [K in keyof T as K]: T[K] };   // adding `as` breaks it in some cases
```

**Why you care:** `Partial<string[]>` gives `(string | undefined)[]`, not a broken object — because `Partial` is homomorphic. If you write a mapped type that mangles arrays, you've accidentally broken homomorphism (usually by mapping over `keyof T & string` or adding an `as` clause).

---

## 3. Key remapping with `as` — where it gets powerful

TypeScript 4.1 added `as` in mapped types, letting you **rename or remove keys**.

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K]
};

type G = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number }
```

### Filtering keys — the pattern you'll use most
**Mapping a key to `never` removes it.**

```ts
// Keep only keys whose value matches V
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K]
};

interface Payment { id: string; money: Money; amountMinor: number; capture(): void }

type Strings = PickByValue<Payment, string>;      // { id: string }
type Methods = PickByValue<Payment, Function>;    // { capture(): void }
type Data    = OmitByValue<Payment, Function>;    // everything except methods

type OmitByValue<T, V> = { [K in keyof T as T[K] extends V ? never : K]: T[K] };
```

```ts
// Split required from optional keys — genuinely useful for form/API types
type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K
}[keyof T];
type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never
}[keyof T];

type R = RequiredKeys<{ a: string; b?: number }>;   // "a"
type O = OptionalKeys<{ a: string; b?: number }>;   // "b"
```
That `{} extends Pick<T, K>` trick is the standard way to detect optionality — `Pick<T, "b">` is `{b?: number}`, and `{}` is assignable to it precisely because `b` is optional.

Note the **map-then-index idiom** again (`{...}[keyof T]`) from [Lesson 10](10-conditional-types-and-infer.md): build an object of keys-or-`never`, then index it to get the union of values.

---

## 4. Template literal types

```ts
type Greeting = `Hello, ${string}`;
const a: Greeting = "Hello, world";     // ✅
const b: Greeting = "Goodbye";          // ❌

type Route = `/v1/${string}`;
type EventName = `payment.${"created" | "succeeded" | "failed"}`;
// "payment.created" | "payment.succeeded" | "payment.failed"
```

**Unions multiply** — a template with two union inputs produces the cross product:
```ts
type Size = "sm" | "md" | "lg";
type Side = "top" | "bottom";
type Class = `p${Side extends "top" ? "t" : "b"}-${Size}`;   // 3 combinations

type Coord = `${"a"|"b"|"c"}${1|2|3}`;   // 9 members: "a1" | "a2" | ... | "c3"
```
⚠️ **This explodes fast.** Four 10-member unions is 10,000 types and the compiler will complain (`Expression produces a union type that is too complex to represent`). Keep the cross product small.

### The four intrinsic string types
```ts
type A = Uppercase<"hello">;     // "HELLO"
type B = Lowercase<"HELLO">;     // "hello"
type C = Capitalize<"hello">;    // "Hello"
type D = Uncapitalize<"Hello">;  // "hello"
```
These are compiler intrinsics — you can't implement them yourself.

### Parsing strings with `infer`
Template literals plus `infer` give you type-level string parsing.

```ts
// Extract path parameters from a route
type PathParams<T extends string> =
  T extends `${string}:${infer P}/${infer Rest}` ? P | PathParams<Rest>
  : T extends `${string}:${infer P}` ? P
  : never;

type P = PathParams<"/v1/merchants/:mid/payments/:pid">;   // "mid" | "pid"

// Now the params object is derived from the route string itself:
type ParamsOf<T extends string> = { [K in PathParams<T>]: string };
type Params = ParamsOf<"/v1/payments/:id">;   // { id: string }
```

```ts
// Split a string
type Split<S extends string, D extends string> =
  S extends `${infer Head}${D}${infer Tail}` ? [Head, ...Split<Tail, D>] : [S];

type Parts = Split<"a.b.c", ".">;    // ["a", "b", "c"]

// Join a tuple
type Join<T extends readonly string[], D extends string> =
  T extends readonly [infer F extends string, ...infer R extends string[]]
    ? R extends [] ? F : `${F}${D}${Join<R, D>}`
    : "";
type J = Join<["a", "b", "c"], "/">;   // "a/b/c"
```

### Type-safe deep property paths
```ts
type Paths<T> = T extends object
  ? { [K in keyof T & string]: T[K] extends object ? K | `${K}.${Paths<T[K]>}` : K }[keyof T & string]
  : never;

type P2 = Paths<{ user: { profile: { name: string } }; id: string }>;
// "user" | "user.profile" | "user.profile.name" | "id"

// And resolve a path back to its value type
type PathValue<T, P extends string> =
  P extends `${infer K}.${infer Rest}`
    ? K extends keyof T ? PathValue<T[K], Rest> : never
    : P extends keyof T ? T[P] : never;

type V = PathValue<{ user: { name: string } }, "user.name">;   // string

declare function get<T, P extends Paths<T>>(obj: T, path: P): PathValue<T, P>;
const name = get({ user: { name: "a" } }, "user.name");   // string ✅
get({ user: { name: "a" } }, "user.nmae");                // ❌ typo caught
```
**This is how `react-hook-form`, `lodash.get`'s types, and i18n libraries achieve typed string paths.** It's also right at the boundary of "worth it" — powerful in a library, usually too clever in app code.

---

## 5. The patterns worth keeping

```ts
// 1. Prettify — flatten ugly intersections in hovers. Keep this in every project.
type Prettify<T> = { [K in keyof T]: T[K] } & {};

// 2. Deep variants, with built-ins excluded (Lesson 10)
type Builtin = string | number | boolean | bigint | symbol | null | undefined
             | Date | RegExp | ((...a: any[]) => any);

type DeepPartial<T> =
  T extends Builtin ? T
  : T extends readonly (infer E)[] ? readonly DeepPartial<E>[]
  : T extends object ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// 3. Require at least one key
type RequireAtLeastOne<T, K extends keyof T = keyof T> =
  Omit<T, K> & { [P in K]-?: Required<Pick<T, P>> & Partial<Omit<T, P>> }[K];

type Filter = RequireAtLeastOne<{ status?: string; customer?: string; created?: string }>;
// {} is rejected; any one key is enough
```
`RequireAtLeastOne` is genuinely useful for API filter objects — *"you must filter by something."*

```ts
// 4. Exactly one key (a tagged-union alternative)
type ExactlyOne<T, K extends keyof T = keyof T> =
  { [P in K]: Required<Pick<T, P>> & Partial<Record<Exclude<K, P>, never>> }[K];

// 5. Mutually exclusive props — the React pattern
type XOR<A, B> =
  | (A & { [K in Exclude<keyof B, keyof A>]?: never })
  | (B & { [K in Exclude<keyof A, keyof B>]?: never });

type LinkOrButton = XOR<{ href: string }, { onClick: () => void }>;
const ok1: LinkOrButton = { href: "/x" };                   // ✅
const bad: LinkOrButton = { href: "/x", onClick: () => {} }; // ❌ can't have both
```
`XOR` solves a real React problem: a component that's *either* a link *or* a button, never both. You'll use it in [Lesson 17](../05-ecosystem/17-typescript-with-react.md).

```ts
// 6. Typed CSS / design tokens
type Space = 0 | 1 | 2 | 4 | 8;
type SpacingClass = `${"m" | "p"}${"" | "t" | "r" | "b" | "l" | "x" | "y"}-${Space}`;
const cls: SpacingClass = "px-4";      // ✅
const bad2: SpacingClass = "px-3";     // ❌ 3 isn't in the scale
```
That's a design system's spacing scale enforced by the compiler — a small, genuinely delightful use.

---

## 6. Ledger's type-level layer

```ts
// ─── Event names, derived from the domain ───────────────────
type Resource = "payment" | "refund" | "payout" | "customer";
type Action   = "created" | "updated" | "succeeded" | "failed";
export type EventName = `${Resource}.${Action}`;               // 16 members
export type EventGlob = EventName | `${Resource}.*` | "*";      // what webhooks subscribe to

// ─── Routes, with params derived from the path string ───────
type PathParams<T extends string> =
  T extends `${string}:${infer P}/${infer R}` ? P | PathParams<R>
  : T extends `${string}:${infer P}` ? P : never;

export type ParamsOf<T extends string> =
  [PathParams<T>] extends [never] ? {} : { [K in PathParams<T>]: string };

type Check = ParamsOf<"/v1/payments/:id">;      // { id: string }
type Check2 = ParamsOf<"/v1/balance">;          // {}   ← note the [never] guard (Lesson 10)

// ─── API shapes derived from the domain, not hand-written ───
export type CreateInput<T> = Prettify<
  Omit<T, "id" | "createdAt" | "updatedAt" | "status" | "version">
>;
export type UpdateInput<T> = Prettify<Partial<CreateInput<T>>>;

// ─── Serialized form: Dates become ISO strings on the wire ──
export type Serialized<T> =
  T extends Date ? IsoDate
  : T extends Money ? { amountMinor: number; currency: Currency }
  : T extends readonly (infer E)[] ? readonly Serialized<E>[]
  : T extends object ? { [K in keyof T]: Serialized<T[K]> }
  : T;
```

**That last one is a genuinely valuable pattern:** your domain model uses `Date` and branded types; the wire format uses ISO strings and plain objects. `Serialized<T>` derives the wire type from the domain type, so they can't drift — which is the type-level version of [API Lesson 11](../../API/02-rest-design/11-versioning-and-evolution.md)'s "one source of truth."

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Map a key to `never` in an `as` clause to remove it** | The idiom for filtering keys |
| **Keep `Prettify<T>` in your utils and wrap public types in it** | Readable hovers and error messages |
| **Preserve homomorphism** (`[K in keyof T]`) unless you need remapping | Or `Partial<string[]>` breaks arrays |
| **Exclude `Date`/`Map`/`Set`/functions from deep mapped types** | Otherwise they get mangled into `{}` shapes |
| **Watch union cross-products in template literals** | Four 10-member unions is 10,000 types and a compiler error |
| **`[T] extends [never]` when a template/mapped type may receive `never`** | Distribution over an empty union yields `never` |
| **Derive wire types from domain types (`Serialized<T>`)** | One source of truth |
| **Derive input types from entities (`CreateInput<T>`)** | Adding a field updates both automatically |
| **Use `XOR` for mutually-exclusive props** | Better than runtime checks and a `!` |
| **Stop at the point you can't explain it in 60 seconds** | Especially in app code |

---

## 8. Interview traps

**Q1. "What's a mapped type?"**
A type that iterates over keys to build a new object type: `{ [K in keyof T]: ... }`. With `?`/`-?` and `readonly`/`-readonly` modifiers. Every utility type (`Partial`, `Readonly`, `Pick`, `Record`) is one.

**Q2. "Implement `Partial`, `Required`, `Readonly` and `Pick`."**
```ts
type Partial2<T>  = { [K in keyof T]?: T[K] };
type Required2<T> = { [K in keyof T]-?: T[K] };
type Readonly2<T> = { readonly [K in keyof T]: T[K] };
type Pick2<T, K extends keyof T> = { [P in K]: T[P] };
```
The `-?` and `-readonly` modifiers are the part people don't know.

**Q3. "How do you remove keys in a mapped type?"**
Key remapping with `as`, mapping unwanted keys to `never`: `{ [K in keyof T as Cond ? K : never]: T[K] }`. That's how `PickByValue` and "omit all methods" are built.

**Q4. "What's a homomorphic mapped type and why does it matter?"**
One of the form `{ [K in keyof T]: ... }` mapping directly over `keyof T`. It preserves `readonly`/`?` modifiers and distributes over arrays, tuples and unions — which is why `Partial<string[]>` is `(string|undefined)[]` and not a broken object. Adding certain `as` clauses breaks it.

**Q5. "What are template literal types for, really?"**
Type-safe strings: event names, route paths with typed params, CSS class scales, i18n keys, `on${Capitalize<Event>}` handler names. Combined with `infer` they give type-level string parsing, which is how typed `get(obj, "a.b.c")` works.

**Q6. "How do you get typed deep property paths?"**
A recursive mapped type producing `K | \`${K}.${Paths<T[K]>}\``, plus a `PathValue<T, P>` that walks the path back down. It's how `react-hook-form` types field names. Add the caveat: powerful in a library, usually too clever in app code.

**Q7. "How would you type a component that takes *either* `href` or `onClick`, never both?"**
`XOR<A, B>` — each branch intersected with `{[K in the other's keys]?: never}`, so supplying both fails. Better than a runtime check because it's enforced at the call site.

**Q8. "What's the risk with template literal unions?"**
Combinatorial explosion. Each union input multiplies, and the checker errors with "union type that is too complex to represent" — and before that, it just gets slow. Keep the cross product small, or use `string` with a runtime check.

**Q9. "How do you derive your API input types from your entity types?"**
`Omit` the server-owned fields: `type CreateInput<T> = Omit<T, "id"|"createdAt"|"status">`, and `UpdateInput<T> = Partial<CreateInput<T>>`. Then adding a field to the entity automatically updates both — and, importantly, the server-owned fields are structurally unable to appear in an input, which is the mass-assignment defence from [API Lesson 09](../../API/02-rest-design/09-writes-patch-and-bulk.md) expressed in types.

---

## 9. Build & break

### Build — implement these
```ts
type MyPartial<T>       = ?;
type MyRequired<T>      = ?;
type MyReadonly<T>      = ?;
type MyPick<T, K>       = ?;
type MyRecord<K, V>     = ?;
type Mutable<T>         = ?;
type Getters<T>         = ?;    // name → getName(): T
type Setters<T>         = ?;    // name → setName(v: T): void
type PickByValue<T, V>  = ?;
type OmitByValue<T, V>  = ?;
type RequiredKeys<T>    = ?;
type OptionalKeys<T>    = ?;
type Prettify<T>        = ?;
type DeepPartial<T>     = ?;    // with Builtin exclusions
type DeepReadonly<T>    = ?;    // ditto
type Paths<T>           = ?;
type PathValue<T, P>    = ?;
type Split<S, D>        = ?;
type XOR<A, B>          = ?;
```

### Build — Ledger's derived types
Write `src/types/derive.ts` with `Prettify`, `CreateInput`, `UpdateInput`, `Serialized`, `EventName`, `ParamsOf`. Then prove they work:
```ts
type _1 = Expect<Equal<CreateInput<Customer>, { email: string; name?: string; phone?: string }>>;
type _2 = Expect<Equal<ParamsOf<"/v1/payments/:id">, { id: string }>>;
type _3 = Expect<Equal<ParamsOf<"/v1/balance">, {}>>;
type _4 = Expect<Equal<Serialized<{ createdAt: Date }>, { createdAt: IsoDate }>>;

// The type-level test harness (from type-challenges) — worth having:
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends (<T>() => T extends Y ? 1 : 2) ? true : false;
type Expect<T extends true> = T;
```
That `Equal` implementation is famous and worth recognising — it exploits the fact that two generic function signatures are only identical if their bodies' conditional types are identical, which is the only reliable way to test type equality.

### Break — five experiments
1. **Homomorphism.** `type Bad<T> = { [K in keyof T & string]: T[K] }` vs `{ [K in keyof T]: T[K] }`. Apply both to `string[]`. One stays an array.
2. **Union explosion.** Build a template literal from four 10-member unions. Read the compiler's complaint.
3. **Deep types vs `Date`.** `DeepPartial<{ created: Date }>` without the `Builtin` guard — hover `created` and see the mangling.
4. **`never` in a mapped type.** `ParamsOf<"/v1/balance">` without the `[never] extends` guard. See what distribution over `never` gives you.
5. **`Prettify`.** Hover `Pick<Payment, "id"> & { extra: boolean }` before and after wrapping in `Prettify`. That readability difference is why you keep it.

### Explain out loud (90 seconds)
1. What a mapped type is, and what `-?` / `-readonly` do.
2. How key remapping removes keys.
3. What homomorphism preserves, and why it matters for arrays.
4. Three real uses of template literal types.
5. When you'd stop.

---

## What's next

You can now build any utility type. Next: the **complete catalogue** of built-in utility types — what each does, which ones have gotchas (`Omit` and `Exclude` both have famous ones), and when to reach for a library instead of writing your own.

Next → **[Lesson 12: The utility types, and building your own](12-utility-types.md)**
