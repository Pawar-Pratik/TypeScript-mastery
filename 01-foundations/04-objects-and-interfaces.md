# Lesson 04 — Objects, interfaces vs types, and the shape rules

> **Why this lesson exists:** object types are 90% of the TypeScript you'll write, and three of their rules confuse essentially everyone exactly once — **excess property checks**, **optional vs `undefined`**, and **`interface` vs `type`**. This lesson settles all three mechanically, and introduces `satisfies`, which is the most useful operator added to the language in years and which most engineers still don't use.

**Time:** ~65 minutes · **Prereq:** Lesson 03

---

## 1. The idea in one sentence

> **An object type is a set of *requirements*, not a description of an exact shape — a value satisfies it by having at least those members, which is why "extra properties are fine" is the default and the exceptions are what need explaining.**

---

## 2. `interface` vs `type` — the real answer

Both describe object shapes. They are **not** interchangeable, and "just pick one" is a weak interview answer. There are five real differences.

### Difference 1 — `type` can describe anything; `interface` only object shapes

```ts
type ID = string | number;                    // ✅ union
type Fn = (a: number) => string;              // ✅ function (interfaces can too, awkwardly)
type Pair = [string, number];                 // ✅ tuple
type Status = "on" | "off";                   // ✅ literals
type Keys = keyof User;                       // ✅ computed
type Partial2<T> = { [K in keyof T]?: T[K] }; // ✅ mapped type

interface ID2 = string | number;              // ❌ syntax error
```
**Anything computed, unioned, or non-object must be a `type`.** That alone settles most cases.

### Difference 2 — declaration merging

```ts
interface Window { myApp: App }     // ✅ MERGES with the global Window
interface Window { version: string } // ✅ merges again

type A = { x: number };
type A = { y: number };              // ❌ Duplicate identifier 'A'
```

This is a genuine feature with a genuine purpose: **augmenting types you don't own.**
```ts
// Extending Express's Request — you cannot do this with a type alias
declare global {
  namespace Express {
    interface Request { principal?: Principal; merchantId?: string; repos?: Repos }
  }
}
```
It's also a hazard in application code: two files can silently merge an interface, producing a type neither author intended. **Feature in `.d.ts`, footgun in `src/`.**

### Difference 3 — `extends` vs `&`, and how errors are reported

```ts
interface A { x: number }
interface B extends A { y: string }       // checked at declaration
type C = A & { x: string };                // NOT checked — you get x: never

// The important difference:
interface Bad extends A { x: string }
// ❌ Interface 'Bad' incorrectly extends 'A'. Type 'string' is not assignable to 'number'
//    ← caught HERE, at the declaration, with a clear message

type AlsoBad = A & { x: string };
const v: AlsoBad = { x: "s" };
// ❌ error is reported at the USE SITE, and says x should be `never`
```
**`extends` fails fast and clearly; `&` fails later and cryptically.** That's a real argument for `interface` when you're modelling an inheritance hierarchy.

### Difference 4 — compiler performance at scale

`interface` creates a named, cached type reference. `&` creates a new anonymous type that must be re-resolved. In a large codebase with deeply nested intersections this is measurable — the TypeScript team's own performance guidance recommends `interface extends` over intersections for this reason. Not a concern at 10 files; a real concern at 10,000 ([Lesson 18](../05-ecosystem/18-tooling-and-performance.md)).

### Difference 5 — `implements`
```ts
class Impl implements MyInterface {}    // ✅
class Impl2 implements MyTypeAlias {}   // ✅ also works, if the alias is an object type
```
Both work; only a union type alias can't be implemented.

### The recommendation

```
Modelling an object shape or a class contract?     → interface
Anything else (union, tuple, function, computed,
mapped, conditional)?                              → type
Augmenting a third-party or global type?           → interface (you need merging)
Public library API you want users to extend?       → interface
Want to prevent accidental merging?                → type
```

> **The interview answer:** *"`type` is strictly more capable — it can express unions, tuples, and computed types that `interface` can't. `interface` has three advantages: declaration merging, which you need to augment types you don't own; `extends` reports conflicts at the declaration with a clear message rather than at the use site as `never`; and named interfaces are cheaper for the compiler at scale. So: `interface` for object shapes, `type` for everything else. In practice consistency matters more than the choice."*

---

## 3. Excess property checks — the rule that confuses everyone once

```ts
interface Options { url: string; timeout?: number }

// ❌ Object literal may only specify known properties, and 'timeoutt' does not exist
const a: Options = { url: "/x", timeoutt: 5000 };

// ✅ NO ERROR — same object, assigned through a variable
const raw = { url: "/x", timeoutt: 5000 };
const b: Options = raw;
```

**Why the difference?** Structural typing says extra properties are fine — `raw` has everything `Options` requires, so it's assignable. That's correct and necessary (otherwise you could never pass a `Dog` where an `Animal` is wanted).

But TypeScript adds a special rule for **fresh object literals**: when you write an object literal *directly* in a position with a known target type, extra properties are flagged. The reasoning is that a literal written inline can only be for that one purpose, so an unexpected property is almost certainly a typo — and typos in optional-property names are otherwise completely silent.

**The check applies to freshness, and freshness is lost on assignment to a variable.** That's the whole mechanism.

```ts
// Freshness is also lost through a function return, a spread, or `as`:
function make(): Options { return { url: "/x", extra: 1 }; }  // ❌ still fresh
const c: Options = { ...raw };                                 // ✅ spread — not fresh
const d: Options = { url: "/x", extra: 1 } as Options;         // ✅ assertion kills the check
```

### Why this matters in real code
```ts
interface ButtonProps { onClick?: () => void; disabled?: boolean }

<Button onclick={fn} />        // ❌ caught! lowercase typo
// Without excess property checks, `onclick` would be silently ignored and the
// button would never respond — a bug with no error anywhere.
```
**This check is why React prop typos are caught**, and it's a strong argument for passing object literals inline rather than through intermediate variables.

### When you genuinely want extra properties
```ts
// 1. An index signature says "and anything else"
interface Options2 { url: string; [key: string]: unknown }

// 2. `satisfies` — checks without widening (see §7). Usually the right answer.
const opts = { url: "/x", timeout: 5000 } satisfies Options;
```

---

## 4. Optional vs `undefined` — three different things

```ts
interface A { x?: number }                    // may be ABSENT
interface B { x: number | undefined }         // MUST be present, may be undefined
interface C { x?: number | undefined }        // may be absent OR undefined

const a1: A = {};                             // ✅
const b1: B = {};                             // ❌ Property 'x' is missing
const b2: B = { x: undefined };               // ✅
```

`B` is the useful and under-used one: **it forces the author to acknowledge the field exists.** With `A`, someone can forget `x` entirely and you'll never know whether that was intentional.

```ts
// A concrete example where B is right:
type UpdateCustomer = {
  name: string | undefined;      // you MUST state whether you're changing it
  phone: string | null | undefined;
};
// The compiler now forces every call site to think about every field.
```

### With `exactOptionalPropertyTypes` on (Lesson 02)
```ts
const a2: A = { x: undefined };   // ❌ under the flag — `x?: number` means absent, not undefined
```
This is the flag that makes "absent" and "explicitly undefined" genuinely different types — which is exactly the PATCH-semantics distinction from [API Lesson 09](../../API/02-rest-design/09-writes-patch-and-bulk.md).

### Detecting absence at runtime
```ts
if ("phone" in patch) { }             // ✅ true even when phone is undefined/null
if (patch.phone !== undefined) { }    // ❌ can't distinguish absent from undefined
if (Object.hasOwn(patch, "phone")) {} // ✅ the modern, safest form
```
**`in` (or `Object.hasOwn`) is how you implement merge-patch semantics.** Truthiness checks lose the distinction, and that's the data-loss bug.

---

## 5. Index signatures and `Record`

```ts
interface Dict { [key: string]: number }
type Dict2 = Record<string, number>;              // identical, and more readable
type ByStatus = Record<PaymentStatus, number>;    // ★ every key REQUIRED — exhaustive
type Partial2 = Partial<Record<PaymentStatus, number>>;  // keys optional
```

`Record<Union, V>` is genuinely valuable: **it forces exhaustiveness on an object**, the same way a `never` check does on a switch.
```ts
const LABELS: Record<PaymentStatus, string> = {
  requires_payment_method: "Awaiting", requires_capture: "Authorised",
  succeeded: "Paid", failed: "Failed",
  // ❌ Property 'refunded' is missing  ← add a status, this breaks. Exactly right.
};
```
Prefer this to a switch when you just need a lookup — it's shorter and equally exhaustive.

### The index-signature lie
```ts
const scores: Record<string, number> = { alice: 10 };
const bob = scores.bob;      // type: number.   Runtime: undefined 💥
```
The type claims every string key yields a `number`. **`noUncheckedIndexedAccess` fixes exactly this**, making it `number | undefined`. It's the flag's headline use case.

### Mixing an index signature with known keys
```ts
interface Config {
  url: string;
  [key: string]: string | number;   // known props must be assignable to this
}
// interface Bad { url: string; retries: boolean; [k: string]: string }  ❌ boolean not assignable
```

### `noPropertyAccessFromIndexSignature`
```ts
const env: Record<string, string> = process.env as any;
env.DATABASE_URL;      // ❌ under the flag — use env["DATABASE_URL"]
```
The point: dot access implies "I know this property exists"; bracket access admits you're doing a dynamic lookup. Worth enabling — it makes the uncertainty visible.

### `Map` vs an object
```ts
const byId = new Map<PaymentId, Payment>();
```
Prefer `Map` when keys are dynamic, numerous, or not strings; when insertion order matters; or when you need `.size`. `Map.get` **correctly returns `V | undefined`** without needing any flag, which is a real advantage.

---

## 6. `readonly`, `keyof`, `typeof` and indexed access

```ts
interface Payment {
  readonly id: string;            // can't reassign after creation
  amountMinor: number;
  readonly metadata: { note: string };   // shallow! metadata.note IS mutable
}
```
**`readonly` is shallow.** Deep immutability needs a recursive mapped type ([Lesson 11](../03-type-level/11-mapped-and-template-literal-types.md)).

```ts
type PaymentKeys = keyof Payment;             // "id" | "amountMinor" | "metadata"
type Amount = Payment["amountMinor"];         // number         ← indexed access
type Note = Payment["metadata"]["note"];      // string         ← nested
type AnyValue = Payment[keyof Payment];       // string | number | { note: string }

const p = { id: "pi_1", amountMinor: 4999 };
type P = typeof p;                             // { id: string; amountMinor: number }
```

**`typeof` (the type operator) is one of the most useful tools in the language** — it lets a value be the single source of truth:

```ts
const DEFAULT_OPTIONS = { retries: 3, timeoutMs: 5000, verbose: false };
type Options = Partial<typeof DEFAULT_OPTIONS>;
// The defaults object and the options type can never drift apart.
```

```ts
// A generic accessor — this pattern is everywhere in real code
function get<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const amt = get(payment, "amountMinor");   // inferred as number, not `any`
get(payment, "nope");                       // ❌ not assignable to keyof Payment
```

---

## 7. `satisfies` — the operator you should be using

Added in TypeScript 4.9, and it solves a problem every codebase has.

**The problem:** annotating loses precision, and not annotating loses checking.

```ts
type Config = Record<string, string | number>;

// Option A: annotate → checked, but precision is LOST
const a: Config = { url: "/api", retries: 3 };
a.url.toUpperCase();     // ❌ 'string | number' has no method 'toUpperCase'
a.nope;                   // ✅ no error — the index signature allows anything

// Option B: no annotation → precise, but UNCHECKED
const b = { url: "/api", retries: 3 };
b.url.toUpperCase();     // ✅ url is string
// but nothing verified that b matches Config at all

// Option C: `satisfies` → checked AND precise  ✅✅
const c = { url: "/api", retries: 3 } satisfies Config;
c.url.toUpperCase();     // ✅ url is string
c.nope;                   // ❌ Property 'nope' does not exist
// and if you wrote `retries: true`, it errors — because it must satisfy Config
```

`satisfies` says *"check this against that type, but keep the narrow inferred type."* That's exactly what you want almost every time you were reaching for an annotation on a constant.

### The two patterns you'll use constantly

```ts
// 1. A config/lookup table: checked for completeness, precise for consumers
const ROUTES = {
  payments: "/v1/payments",
  refunds:  "/v1/refunds",
  balance:  "/v1/balance",
} satisfies Record<string, `/v1/${string}`>;

type RouteName = keyof typeof ROUTES;         // "payments" | "refunds" | "balance"
const url = ROUTES.payments;                   // type: "/v1/payments"  ← LITERAL, not string

// 2. Exhaustiveness on a lookup, with precise values retained
const LABELS = {
  succeeded: "Paid", failed: "Failed", refunded: "Refunded",
  requires_capture: "Authorised", requires_payment_method: "Awaiting",
} satisfies Record<PaymentStatus, string>;
// Add a status → error here. And LABELS.succeeded is "Paid", not string.
```

**When to use which:**

| Goal | Use |
|---|---|
| Constrain a value **and** keep its literal type | **`satisfies`** |
| Declare a variable that will hold various values of a type | annotation (`const x: T = …`) |
| Override the compiler because you know better | `as` — and comment why |
| Prevent widening + make readonly | `as const` (combine: `... as const satisfies T`) |

> `as const satisfies T` is a genuinely useful combination: deeply-readonly, non-widened, *and* validated against `T`.

---

## 8. Modelling shapes precisely

```ts
// Recursive types — for JSON, trees, nested comments
type Json = string | number | boolean | null | Json[] | { [k: string]: Json };

type TreeNode<T> = { value: T; children: TreeNode<T>[] };

// Function and constructor members
interface Handler {
  (event: Event): void;                 // callable
  new (x: number): Handler;             // constructible
  readonly name: string;                // and properties
}

// Method shorthand vs property — NOT equivalent, and it matters
interface A { compare(a: string, b: string): number }        // method: BIVARIANT params
interface B { compare: (a: string, b: string) => number }    // property: CONTRAVARIANT params
```
That last distinction is the deliberate unsoundness from [Lesson 01](01-why-typescript-and-erasure.md), and it gets its full explanation in [Lesson 06](../02-type-system/06-structural-typing-and-variance.md). For now: **method shorthand is less safe than the property form**, and `strictFunctionTypes` only applies to the property form.

### Ledger's core types
```ts
// Discriminated union — the shape that makes illegal states impossible (Lesson 09)
type Payment =
  | { status: "requires_payment_method"; id: PaymentId; amountMinor: number; currency: Currency }
  | { status: "requires_capture";        id: PaymentId; amountMinor: number; currency: Currency;
      authorizedAt: string }
  | { status: "succeeded";               id: PaymentId; amountMinor: number; currency: Currency;
      capturedAt: string }
  | { status: "failed";                  id: PaymentId; amountMinor: number; currency: Currency;
      failureCode: FailureCode };

// Note what this buys: `capturedAt` is only accessible after you've narrowed to
// "succeeded". You cannot read a capture timestamp off a failed payment — it isn't
// in the type. That's the difference between "validated" and "unrepresentable".
```

---

## 9. Production rules

| Rule | Why |
|---|---|
| **`interface` for object shapes, `type` for everything else** | `type` is more capable; `interface` gives better errors, merging and compiler performance |
| **Never merge interfaces in application code** | Two files can silently combine into a type neither author intended |
| **Pass object literals inline where possible** | Keeps excess property checks active — that's what catches prop typos |
| **`satisfies` for every constant you'd have annotated** | Checked *and* precise |
| **`as const satisfies T` for config tables** | Readonly + literal + validated |
| **`Record<Union, V>` for lookups over a union** | Forces exhaustiveness; adding a member is a compile error |
| **`noUncheckedIndexedAccess` on, or prefer `Map`** | Index signatures lie about presence; `Map.get` tells the truth |
| **Use `in` / `Object.hasOwn` to detect absence, not truthiness** | The absent-vs-undefined distinction is real, and losing it loses data |
| **`x: T \| undefined` when the author must acknowledge the field** | `x?: T` lets them forget it silently |
| **`readonly` on fields the owner controls** | Documents intent even though it's erased. Remember it's shallow |
| **Prefer property-style function members over method shorthand** | Method shorthand is bivariant, i.e. less safe |

---

## 10. Interview traps

**Q1. "`interface` vs `type`?"**
Give the five differences (§2), and lead with capability: `type` can express unions/tuples/computed types; `interface` gives declaration merging, declaration-site conflict errors, and better compiler performance at scale. End with your rule and note that consistency matters more than the choice.

**Q2. "Why does this error but assigning through a variable doesn't?"**
Excess property checks on **fresh object literals**. Structural typing allows extra properties, but a literal written inline can only be for that one target, so an unexpected key is almost certainly a typo. Freshness is lost via a variable, a spread or `as`. **This is what catches React prop typos.**

**Q3. "`x?: number` vs `x: number | undefined`?"**
The first may be absent; the second must be present but may be `undefined`. Use the second when you want to force callers to acknowledge the field. With `exactOptionalPropertyTypes`, `x?: number` also forbids explicitly passing `undefined` — which is what makes merge-patch semantics expressible.

**Q4. "What's wrong with `Record<string, T>`?"**
It claims every string key yields a `T`, so `dict.missing` types as `T` and is `undefined` at runtime. Fix: `noUncheckedIndexedAccess`, or use a `Map`, whose `.get` returns `V | undefined` by construction.

**Q5. "What does `satisfies` do and why not just annotate?"**
It validates a value against a type **without widening it**. Annotating widens (so you lose literal types and gain the index signature's permissiveness); omitting the annotation keeps precision but checks nothing. `satisfies` gives both. It's the correct tool for config objects, route maps and lookup tables.

**Q6. "How do you make a lookup object exhaustive over a union?"**
`Record<TheUnion, V>` — every key becomes required, so adding a union member is a compile error at the object literal. Combine with `satisfies` to keep the precise value types.

**Q7. "Is `readonly` enforced at runtime?"**
No — erased. And it's **shallow**: `readonly metadata: { note: string }` still allows `p.metadata.note = "x"`. Deep immutability needs a recursive mapped type, and `Object.freeze` for runtime.

**Q8. "How do you type a recursive JSON value?"**
```ts
type Json = string | number | boolean | null | Json[] | { [k: string]: Json };
```
Type aliases may be recursive when the recursion is inside an object or array. Follow-up worth pre-empting: this is why `JSON.parse` returns `any` rather than `Json` — `any` is assignable to anything, which the standard library chose for compatibility, and it's a good argument for `ts-reset` ([Lesson 19](../05-ecosystem/19-migration-and-strictness.md)).

**Q9. "How would you type `Express.Request` with your own fields?"**
`declare global { namespace Express { interface Request { … } } }` — using **declaration merging**, which is the concrete reason `interface` still matters and something a `type` alias cannot do.

**Q10. "`keyof` and `typeof` — what are they for?"**
`keyof T` gives the union of `T`'s keys; `typeof value` gives the type of a runtime value. Together they let a *value* be the source of truth: `const DEFAULTS = {...}; type Options = Partial<typeof DEFAULTS>` — so the defaults and the type can never drift.

---

## 11. Build & break

### Build — Ledger's shared type layer (start it now; you'll extend it all track)
`src/types/ledger.ts`:
```ts
// Derive the union from the runtime list — one source of truth
export const CURRENCIES = ["usd", "eur", "gbp", "inr", "jpy"] as const;
export type Currency = typeof CURRENCIES[number];

export const PAYMENT_STATUSES = [
  "requires_payment_method", "requires_capture", "succeeded", "failed", "refunded",
] as const;
export type PaymentStatus = typeof PAYMENT_STATUSES[number];

// Minor-unit exponents: exhaustive over Currency, precise values retained
export const MINOR_UNITS = {
  usd: 2, eur: 2, gbp: 2, inr: 2, jpy: 0,
} as const satisfies Record<Currency, number>;
// ← add a currency to CURRENCIES and THIS line errors. Exactly what you want.

export interface Money {
  readonly amountMinor: number;
  readonly currency: Currency;
}

export const STATUS_LABELS = {
  requires_payment_method: "Awaiting payment method",
  requires_capture: "Authorised",
  succeeded: "Paid",
  failed: "Failed",
  refunded: "Refunded",
} as const satisfies Record<PaymentStatus, string>;

// Patch semantics, expressed in types (needs exactOptionalPropertyTypes)
export interface CustomerPatch {
  name?: string;                  // absent = unchanged
  email?: string;                 // absent = unchanged
  phone?: string | null;          // absent = unchanged, null = CLEAR
}
```
Then **add `"chf"` to `CURRENCIES`** and watch exactly one line fail, telling you what to update. That's the mechanism you're building for.

### Break — five experiments
1. **Excess property checks.** Write the literal directly (errors), then via a variable (no error), then via a spread (no error), then with `satisfies` (errors). Four behaviours, one object — write down the rule that explains all four.
2. **The `Record` lie.** With `noUncheckedIndexedAccess` off, `const m: Record<string, number> = {}; m.x.toFixed(2)`. Compiles, crashes. Turn the flag on.
3. **Interface merging.** Declare `interface Config { a: string }` in two files in the same module scope, then in the same file. Note when it merges and when a `type` errors instead.
4. **Shallow readonly.** `readonly meta: { note: string }` — then mutate `p.meta.note`. It compiles. Now write a `DeepReadonly<T>` (or peek at Lesson 11).
5. **Absent vs undefined.** Write a merge-patch function using `patch.phone !== undefined` and then using `"phone" in patch`. Call it with `{}` and with `{ phone: null }`. Only one implementation can clear the field.

### Explain out loud (90 seconds)
1. `interface` vs `type` — three real differences and your rule.
2. Excess property checks: the mechanism and what "fresh" means.
3. The three optional/undefined variants and when each is right.
4. What `satisfies` gives you that an annotation doesn't.
5. Why `Record<Union, V>` is a better lookup than an annotated plain object.

---

## Module 1 complete — checkpoint

Cold, no notes:

- [ ] What TypeScript does at runtime, and the constructs that *do* emit code
- [ ] The five soundness holes
- [ ] Where the compiler's guarantees begin and end (draw the boundary)
- [ ] The eight `strict` flags, and the two that should be in it
- [ ] `target` vs `lib`, and the trap
- [ ] Types as sets; why `never` is assignable to everything
- [ ] Why `let` widens and `const` doesn't; what `as const` fixes
- [ ] `any` vs `unknown` vs `never` vs `void`
- [ ] Why you'd choose a literal union over an `enum`
- [ ] `interface` vs `type`, with three real differences
- [ ] Excess property checks, and why a variable bypasses them
- [ ] `x?: T` vs `x: T | undefined`, and how to detect absence at runtime
- [ ] What `satisfies` does and when to reach for it

---

## What's next

Module 2 is the engine room. It starts with functions — parameters, overloads, contextual typing — and then the single most important lesson in the track: **structural typing and variance**, which explains every "why does this unsafe code compile?" question you will ever have.

Next → **[Lesson 05: Functions, overloads & contextual typing](../02-type-system/05-functions-and-overloads.md)**
