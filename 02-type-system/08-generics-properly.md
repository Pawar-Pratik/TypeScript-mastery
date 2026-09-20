# Lesson 08 — Generics, properly

> **Why this lesson exists:** generics are where TypeScript stops being annotations and becomes a small programming language. Most engineers use them (`Array<T>`, `useState<T>`) without knowing the inference rules, so *"why did it infer `string` instead of `'a' | 'b'`?"* feels arbitrary. It isn't. This lesson gives you the rules — plus the more valuable skill of knowing when a generic is the **wrong** tool, which is more often than people think.

**Time:** ~75 minutes · **Prereq:** Lessons 06, 07

---

## 1. The idea in one sentence

> **A generic exists to preserve a *relationship* between types — if a type parameter appears only once in a signature, it isn't preserving anything and shouldn't be there.**

That test — *"does this type parameter appear at least twice?"* — resolves most "should this be generic?" questions instantly.

---

## 2. Why generics: the relationship

```ts
// ❌ any — loses everything
function first(arr: any[]): any { return arr[0]; }
const n = first([1, 2, 3]);        // any — no help downstream

// ❌ unknown — safe but useless here
function first2(arr: unknown[]): unknown { return arr[0]; }
const n2 = first2([1, 2, 3]);      // unknown — must narrow, pointlessly

// ✅ generic — preserves the relationship between input and output
function first3<T>(arr: readonly T[]): T | undefined { return arr[0]; }
const n3 = first3([1, 2, 3]);      // number | undefined
const s3 = first3(["a", "b"]);     // string | undefined
```

**The relationship is the point:** "whatever's in the array is what comes out." `any` and `unknown` both destroy it.

### The test: does the parameter appear twice?

```ts
// ❌ Pointless generic — T appears once. This is just `(x: unknown) => void`.
function log<T>(x: T): void { console.log(x); }
// ✅
function log(x: unknown): void { console.log(x); }

// ❌ Worse than pointless — this is a disguised `as`
function parse<T>(json: string): T { return JSON.parse(json); }
const p = parse<Payment>(raw);      // ZERO validation happened. Same as `JSON.parse(raw) as Payment`
// ✅
function parse(json: string): unknown { return JSON.parse(json); }
// then validate — Lesson 15

// ✅ Genuine generic — T appears in the parameter AND the return
function identity<T>(x: T): T { return x; }
function pluck<T, K extends keyof T>(items: readonly T[], key: K): T[K][] {
  return items.map(i => i[key]);
}
```

> **`function parse<T>(s: string): T` is the most common bad generic in the wild.** It looks type-safe and is a pure lie: the caller picks `T`, nothing verifies it, and the compiler now believes an unvalidated shape. **Recognising it is a genuinely valuable review skill** — and it's exactly the `as User` problem from [Lesson 01](../01-foundations/01-why-typescript-and-erasure.md) wearing a costume.

---

## 3. Constraints

```ts
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest("abc", "de");             // ✅ string
longest([1, 2], [3]);              // ✅ number[]
longest(1, 2);                     // ❌ number has no 'length'
```

`extends` here means "assignable to", i.e. "at least this shape."

### The `keyof` constraint — the most useful pattern in the language

```ts
function get<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const amt = get(payment, "amountMinor");     // number
get(payment, "nope");                         // ❌ not assignable to keyof Payment

// Multiple keys
function pick<T, K extends keyof T>(obj: T, keys: readonly K[]): Pick<T, K> {
  return Object.fromEntries(keys.map(k => [k, obj[k]])) as Pick<T, K>;
}
const slim = pick(payment, ["id", "amountMinor"]);   // { id: PaymentId; amountMinor: number }
```

### Constraining to a subset of keys by value type
```ts
// Only keys whose value is a string — needs a mapped type (Lesson 11)
type StringKeys<T> = { [K in keyof T]: T[K] extends string ? K : never }[keyof T];

function upper<T, K extends StringKeys<T>>(obj: T, key: K): string {
  return String(obj[key]).toUpperCase();
}
upper(payment, "id");            // ✅
upper(payment, "amountMinor");   // ❌ not a string key
```

### Defaults
```ts
type ApiResult<T = unknown, E = ApiError> =
  | { ok: true; data: T }
  | { ok: false; error: E };

type A = ApiResult;                    // ApiResult<unknown, ApiError>
type B = ApiResult<Payment>;           // ApiResult<Payment, ApiError>

// Defaults may reference earlier parameters:
interface Store<T, K extends keyof T = keyof T> { }
```
**Defaults are for callers; constraints are for you.** A common mistake is using a default where you meant a constraint:
```ts
function f<T = string>(x: T) {}     // ❌ T defaults to string but accepts ANYTHING
f(42);                               // ✅ compiles — T inferred as number
function g<T extends string>(x: T) {}  // ✅ actually restricts
```

---

## 4. Inference: the rules

This is the section that removes the mystery.

### Where inference happens
TypeScript infers type arguments from **argument positions**, matching the parameter types against the actual argument types.

```ts
function f<T>(x: T, y: T): T { return x; }
f("a", "b");            // T = string
f("a", 1);              // T = string | number  (TS unions the candidates)
```

### Inference priority
When a parameter appears in several positions, candidates are collected and TypeScript picks the best one. **Return-type positions are lower priority than parameter positions:**

```ts
function make<T>(factory: () => T, fallback: T): T { return fallback; }
make(() => "a", "b");   // T = string
```

### Widening in generics — the most common surprise
```ts
function tag<T>(value: T): { value: T } { return { value }; }

tag("hello");           // { value: string }    ← WIDENED
tag("hello" as const);  // { value: "hello" }

// The rule: an inferred type parameter widens literal types...
// UNLESS the parameter is constrained to something literal-ish:
function tag2<T extends string>(value: T): { value: T } { return { value }; }
tag2("hello");          // { value: "hello" }   ← NOT widened!
```

**Why:** with no constraint, TypeScript assumes you want the general type. With `extends string`, it infers the most specific type satisfying it. **This is the single most useful inference trick to know:**

```ts
// ❌ Loses the literal
function route<T>(path: T) { return path; }
const r = route("/payments");                   // string

// ✅ Keeps it
function route2<T extends string>(path: T) { return path; }
const r2 = route2("/payments");                  // "/payments"
```

### `const` type parameters (TypeScript 5.0)
```ts
function list<T>(items: T[]): T[] { return items; }
list(["a", "b"]);                         // string[]

function list2<const T>(items: T[]): T[] { return items; }
list2(["a", "b"]);                        // readonly ["a", "b"]   ✅ no `as const` needed
```
`const T` infers as if the caller had written `as const`. **This is a big ergonomic win for library APIs** — it means users don't need to remember `as const`:
```ts
// Before: users had to write `as const`
declare function defineRoutes<T extends Record<string, string>>(r: T): T;
const routes = defineRoutes({ home: "/" } as const);

// After:
declare function defineRoutes2<const T extends Record<string, string>>(r: T): T;
const routes2 = defineRoutes2({ home: "/" });      // literals preserved
```

### When inference fails, and what to do
```ts
// 1. Nothing to infer from → falls back to the constraint or unknown
function make<T>(): T { return null as any; }
const x = make();                     // unknown
const y = make<Payment>();            // explicit

// 2. Conflicting candidates
function f<T>(a: T, b: T): T { return a; }
f({ x: 1 }, { y: 2 });                // T = { x: number } | { y: number }

// 3. Partial inference is all-or-nothing
function g<A, B>(a: A): B { return null as any; }
g<string>("x");                        // ❌ Expected 2 type arguments
// Workaround: split into curried functions
const g2 = <A,>(a: A) => <B,>(): B => null as any;
```

### Debugging inference — the practical technique
```ts
// Trick 1: hover. Always. It tells you exactly what was inferred.

// Trick 2: force the compiler to print a type in an error message
type Debug<T> = { _: T } & string;      // intentionally invalid → error shows T expanded

// Trick 3: assert an expected type and read the failure
const _check: string = someInferredValue;   // error message names the actual type

// Trick 4: expand a type for readable hovers — genuinely useful
type Prettify<T> = { [K in keyof T]: T[K] } & {};
type Ugly = Pick<Payment, "id"> & { extra: boolean };
type Nice = Prettify<Ugly>;      // hover shows the flattened object, not the intersection
```
`Prettify` is worth keeping in every project — it turns unreadable intersection hovers into flat object types.

---

## 5. Generic narrowing: the intersection surprise

From [Lesson 07](07-narrowing-and-control-flow.md), stated properly:

```ts
function f<T extends string | number>(x: T) {
  if (typeof x === "string") {
    x.toUpperCase();          // ✅ x: T & string
    const back: T = x;        // ❌ T & string is not assignable to T
  }
}
```

**Why:** `T` might be a *subtype*, e.g. `"a" | 5`. Narrowing to `string` gives `T & string`, which isn't provably `T`.

**The fix: don't be generic when you don't need to be.**
```ts
// ✅ if you only consume the value, a union parameter is correct and simpler
function f(x: string | number) {
  if (typeof x === "string") { const back: string | number = x; }   // ✅ fine
}
```
This is the §2 test again: if `T` appears once, you didn't need a generic, and you've bought yourself the intersection problem for nothing.

---

## 6. Generic classes, interfaces and the patterns you'll actually write

```ts
// A typed event emitter — the canonical "worth it" generic
type EventMap = {
  "payment.succeeded": { id: PaymentId; money: Money };
  "payment.failed":    { id: PaymentId; code: FailureCode };
  "webhook.delivered": { deliveryId: string; attempt: number };
};

class Emitter<M extends Record<string, unknown>> {
  private handlers: { [K in keyof M]?: Array<(p: M[K]) => void> } = {};

  on<K extends keyof M>(event: K, fn: (payload: M[K]) => void): () => void {
    (this.handlers[event] ??= []).push(fn);
    return () => { this.handlers[event] = this.handlers[event]?.filter(h => h !== fn); };
  }

  emit<K extends keyof M>(event: K, payload: M[K]): void {
    this.handlers[event]?.forEach(h => h(payload));
  }
}

const bus = new Emitter<EventMap>();
bus.on("payment.succeeded", p => p.money.amountMinor);   // ✅ payload fully typed
bus.emit("payment.failed", { id, code: "card_declined" }); // ✅ checked
bus.emit("payment.failed", { id });                        // ❌ missing `code`
bus.on("nope", () => {});                                  // ❌ unknown event
```
**This pattern — a map type plus `K extends keyof M`** — is how every typed emitter, router, state machine and RPC client is built. Learn it; you'll reach for it constantly.

```ts
// Signature-preserving wrappers (Lesson 05)
function memoize<A extends unknown[], R>(fn: (...args: A) => R): (...args: A) => R {
  const cache = new Map<string, R>();
  return (...args) => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key)!;
  };
}

// A generic Result type — Lesson 16
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return r.ok ? { ok: true, value: fn(r.value) } : r;
}

// A fluent builder that tracks what's been set — advanced but genuinely useful
class QueryBuilder<T, Selected extends keyof T = never> {
  select<K extends keyof T>(...keys: K[]): QueryBuilder<T, Selected | K> {
    return this as any;
  }
  build(): Pick<T, Selected> { return {} as any; }
}
const q = new QueryBuilder<Payment>().select("id", "status").build();
//    ^? Pick<Payment, "id" | "status">   — the builder remembers
```

---

## 7. When NOT to use a generic

| Anti-pattern | Why it's wrong | Instead |
|---|---|---|
| `function log<T>(x: T)` | `T` appears once — no relationship preserved | `(x: unknown)` |
| `function parse<T>(s: string): T` | A disguised `as` with zero validation | `unknown` + a schema |
| `function get<T>(url: string): Promise<T>` | Same — the caller invents the type | Validate the response |
| Generic with a single call site | Complexity for no reuse | A concrete type |
| `<T extends any>` | Meaningless constraint | Drop the constraint |
| A generic to avoid writing a union | Harder to read, worse errors | The union |
| Deeply nested generic type gymnastics in app code | Unreadable, slow to compile, hard to debug | A simpler model, or two concrete types |

> **The judgement to express in an interview:** *"generics are for preserving relationships between types. If a type parameter appears once, it's noise or a lie. And there's a readability ceiling — three nested conditional types in application code is usually a sign the domain model is wrong, not that I need cleverer types. In a library the calculus differs, because you're paying complexity once for many consumers."*

---

## 8. Production rules

| Rule | Why |
|---|---|
| **A type parameter must appear at least twice** | Otherwise it preserves no relationship |
| **Never `function parse<T>(s: string): T`** | It's `as` in disguise; the caller asserts an unvalidated shape |
| **`T extends string` (not bare `T`) when you want literals preserved** | Unconstrained parameters widen |
| **`const T` for library APIs that take literal config** | Callers don't need to remember `as const` |
| **Constrain, don't default, when you mean to restrict** | `<T = string>` restricts nothing |
| **`K extends keyof T` for anything key-driven** | The most useful constraint in the language |
| **Prefer a union parameter over a generic when you only consume the value** | Avoids the `T & string` intersection problem |
| **Keep `Prettify<T>` in your utils** | Readable hovers for intersections |
| **Hover to verify inference; don't assume** | Inference has rules, and reading the result is faster than reasoning |
| **Cap type-level complexity in application code** | Three nested conditionals means rethink the model |

---

## 9. Interview traps

**Q1. "When would you use a generic?"**
To preserve a relationship between types — input to output, or between parameters. **State the test:** if the type parameter appears only once in the signature, it isn't preserving anything and should be `unknown` or a concrete type.

**Q2. "What's wrong with `function fetchJson<T>(url: string): Promise<T>`?"**
It's `as` in disguise. The caller picks `T`, nothing validates it, and the compiler now trusts an unverified shape — so the error surfaces far from the fetch. Return `Promise<unknown>` and validate with a schema, deriving the type from the validator so they can't diverge.

**Q3. "Why does `f('hello')` infer `string` instead of `'hello'`?"**
Unconstrained type parameters widen literal types. Constrain it — `<T extends string>` — and TypeScript infers the most specific type satisfying the constraint. Or use `const T` (TS 5.0) to infer as if `as const` were written.

**Q4. "What does `const T` do?"**
Infers literal/readonly types from the call site without the caller writing `as const`. Big ergonomic win for config-taking library functions.

**Q5. "`<T extends keyof U>` — what's it for?"**
Constraining a key parameter to real keys of an object, so `obj[key]` types as `U[T]` instead of `any`, and a typo is a compile error. It's the foundation of `get`, `pick`, typed emitters, and typed routers.

**Q6. "Why can't I assign back to `T` after narrowing it?"**
Narrowing a type parameter yields `T & string`, not `string`, because `T` could be a subtype with extra structure. If you hit this, you probably didn't need a generic — a union parameter is simpler and works.

**Q7. "Difference between a constraint and a default?"**
A constraint (`<T extends string>`) restricts what `T` can be. A default (`<T = string>`) only supplies a value when inference has nothing to work from — it restricts nothing. Confusing them is a common bug.

**Q8. "How do you debug a generic that infers the wrong thing?"**
Hover first. Then assert the expected type and read the error message (it names the actual type). `Prettify<T>` to flatten unreadable intersections. And check whether the parameter is unconstrained (widening) or appears in multiple positions (candidate union).

**Q9. "When are generics the wrong tool?"**
Single call site; type parameter appearing once; using them to avoid a union; and deep type-level gymnastics in application code — where they hurt readability, compile time and debuggability. Libraries have a different cost/benefit, because complexity is paid once and amortised over many consumers.

**Q10. "Design a typed event emitter."**
The §6 pattern: an event-map type, `on<K extends keyof M>(e: K, fn: (p: M[K]) => void)`, `emit<K extends keyof M>(e: K, p: M[K])`. Mention that unknown event names and wrong payload shapes both become compile errors, and that the handler store is a mapped type over `M`.

---

## 10. Build & break

### Build — Ledger's typed API client (you'll use this in the React track)
```ts
// The route map is the single source of truth
type Routes = {
  "GET /v1/payments":       { query: ListQuery;  response: Paginated<Payment> };
  "GET /v1/payments/:id":   { params: { id: PaymentId }; response: Payment };
  "POST /v1/payments":      { body: CreatePayment; response: Payment; idempotent: true };
  "POST /v1/payments/:id/refunds": {
    params: { id: PaymentId }; body: CreateRefund; response: Refund; idempotent: true };
  "GET /v1/balance":        { response: Balance };
};

type HasBody<R>   = R extends { body: infer B } ? { body: B } : {};
type HasParams<R> = R extends { params: infer P } ? { params: P } : {};
type HasQuery<R>  = R extends { query: infer Q } ? { query?: Q } : {};
type NeedsKey<R>  = R extends { idempotent: true } ? { idempotencyKey: string } : {};

export async function api<K extends keyof Routes>(
  route: K,
  opts: HasBody<Routes[K]> & HasParams<Routes[K]> & HasQuery<Routes[K]> & NeedsKey<Routes[K]>,
): Promise<Routes[K]["response"]> {
  /* build the URL from route + params, attach headers, parse and validate */
}

// Now all of this is checked:
const p  = await api("GET /v1/payments/:id", { params: { id } });        // Payment
const l  = await api("GET /v1/payments", { query: { limit: 20 } });      // Paginated<Payment>
const r  = await api("POST /v1/payments", { body, idempotencyKey: k });  // Payment
await api("POST /v1/payments", { body });        // ❌ missing idempotencyKey
await api("GET /v1/payments/:id", {});           // ❌ missing params
await api("GET /v1/nope", {});                   // ❌ unknown route
```
**This is the payoff of Module 2.** The compiler now enforces the API contract from the API track — including *"idempotency key required on money-moving POSTs"*, expressed as a type. (The conditional types are Lesson 10; skim them now, come back after.)

### Build — the generic exercises
Implement these without looking anything up:
```ts
function first<T>(arr: readonly T[]): T | undefined;
function last<T>(arr: readonly T[]): T | undefined;
function pluck<T, K extends keyof T>(items: readonly T[], key: K): T[K][];
function groupBy<T, K extends string | number>(items: readonly T[], fn: (x: T) => K): Record<K, T[]>;
function indexBy<T, K extends keyof T>(items: readonly T[], key: K): Map<T[K], T>;
function unique<T, K>(items: readonly T[], by?: (x: T) => K): T[];
function partition<T>(items: readonly T[], pred: (x: T) => boolean): [T[], T[]];
function zip<A, B>(a: readonly A[], b: readonly B[]): Array<[A, B]>;
function memoize<A extends unknown[], R>(fn: (...a: A) => R): (...a: A) => R;
function once<A extends unknown[], R>(fn: (...a: A) => R): (...a: A) => R;
```
For each: hover the result at a call site and confirm the type is as precise as possible. `groupBy(payments, p => p.status)` should give `Record<PaymentStatus, Payment[]>`, not `Record<string, Payment[]>` — if it doesn't, your constraint is wrong.

### Break — five experiments
1. **The widening surprise.** `function f<T>(x: T) { return x; }` vs `<T extends string>`. Call both with `"hello"` and hover. Then add `const T` and compare again.
2. **The fake generic.** Write `function parse<T>(s: string): T`, call `parse<Payment>('{"id":42}')`, then `.id.toUpperCase()`. Compiles, crashes. That's the pattern to recognise in reviews.
3. **The narrowing intersection.** `function f<T extends string|number>(x: T)` — narrow with `typeof` and try to assign back to `T`. Read the error, then rewrite it as a union parameter.
4. **Constraint vs default.** `function f<T = string>(x: T)` — call it with a number. It compiles. Then use `extends` and watch it fail.
5. **Inference conflict.** `function f<T>(a: T, b: T): T` called with `("a", 1)`. Hover `T`. Then constrain it to `string` and read the error.

### Explain out loud (90 seconds)
1. Why generics exist, and the appears-twice test.
2. Why `parse<T>(s: string): T` is dangerous.
3. Why unconstrained parameters widen, and two ways to keep literals.
4. Why narrowing a type parameter gives an intersection.
5. The typed-emitter pattern, and where else you'd use it.

---

## Module 2 complete — checkpoint

Cold, no notes:

- [ ] Contextual typing: what it is and when it fails
- [ ] Why fewer parameters is allowed, and the `parseInt` consequence
- [ ] When an overload is right, and the three things to try first
- [ ] The signature-preserving wrapper generic
- [ ] Structural vs nominal, and why TypeScript is structural
- [ ] Covariance vs contravariance, with the `Handler<Animal>` reasoning
- [ ] Why method parameters are bivariant, and what it costs
- [ ] Why `Dog[]` → `Animal[]` is unsound and `readonly` fixes it
- [ ] Branding for nominal typing
- [ ] The seven narrowing mechanisms
- [ ] Why discriminated unions beat optional-field soup (8 vs 3)
- [ ] Three limits of narrowing and their fixes
- [ ] Exhaustiveness via `assertNever`
- [ ] The appears-twice test for generics
- [ ] Why `parse<T>` is `as` in disguise
- [ ] Widening in generics, and `T extends string` / `const T`

---

## What's next

Module 3 is where TypeScript becomes a language you *compute* in. It starts with the highest-value application of everything so far: **designing types so that wrong states can't be written down** — branded types, discriminated domains, and `Result`.

Next → **[Lesson 09: Making illegal states unrepresentable](../03-type-level/09-illegal-states-unrepresentable.md)**
