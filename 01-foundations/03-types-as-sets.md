# Lesson 03 — Types as sets: primitives, literals, unions, `any`/`unknown`/`never`

> **Why this lesson exists:** there's one mental model that makes TypeScript predictable instead of mysterious — **a type is a set of values, and "assignable" means "subset of".** With it, unions, intersections, `never`, `unknown`, narrowing and every "why is this assignable?" question become derivable rather than memorised. Without it you're guessing forever.

**Time:** ~60 minutes · **Prereq:** Lesson 02

---

## 1. The idea in one sentence

> **A type is the set of all values that inhabit it; `A` is assignable to `B` if and only if every value in `A` is also in `B`.**

That's it. Everything below is a consequence.

```ts
type A = "red" | "blue";              // the set { "red", "blue" }
type B = "red" | "blue" | "green";    // the set { "red", "blue", "green" }

let b: B = someA;   // ✅ every A value is a B value  (A ⊆ B)
let a: A = someB;   // ❌ "green" is a B but not an A
```

**Wider = bigger set = fewer guarantees.** Narrowing is shrinking the set. That's why the operation is called *narrowing*, and why `narrow → wide` assignment always works while `wide → narrow` doesn't.

---

## 2. The hierarchy

```
                      unknown                     ← the universal set: every value
                         │
     ┌───────────┬───────┴────┬──────────┬─────────────┐
   string      number      boolean     object       symbol/bigint
     │            │           │           │
  "red" "a" …  1 2 3 …    true false   {…} […] …     ← literal types: single-value sets
     │            │           │           │
     └────────────┴─────┬─────┴───────────┘
                        │
                      never                       ← the empty set: no value at all

  any  ← outside the lattice entirely. Assignable BOTH ways. It's an opt-out.
  null / undefined  ← their own single-value sets (with strictNullChecks on)
```

Reading it:
- **`unknown` is the top type.** Everything is assignable *to* it; it's assignable *from* everything and *to* nothing (except `any`/`unknown`).
- **`never` is the bottom type.** Nothing is assignable *to* it; it's assignable *to everything* (vacuously — "every value in the empty set is also a `string`" is trivially true).
- **`any` breaks the lattice.** It's assignable in both directions, which is exactly why it's dangerous.

---

## 3. Literal types and widening

```ts
const a = "red";        // type: "red"      ← literal type, a 1-element set
let   b = "red";        // type: string     ← WIDENED
const c: string = "red";// type: string     ← you asked for the wide type
```

**Why the difference:** `let` can be reassigned, so TypeScript widens the literal to its base type to keep future assignments legal. `const` can't be reassigned, so the narrow type is kept.

This causes the single most common beginner confusion:

```ts
type Method = "GET" | "POST";
function request(m: Method) {}

const m1 = "GET";
request(m1);            // ✅ m1 is "GET"

let m2 = "GET";
request(m2);            // ❌ Argument of type 'string' is not assignable to 'Method'

// Three fixes, in order of preference:
let m3: Method = "GET";           // ✅ annotate — best, expresses intent
request("GET" as const);          // ✅ const assertion
request("GET" as Method);         // ⚠️ works, but assertions are a last resort
```

### Widening in objects — the one that catches everyone

```ts
const config = { method: "GET", retries: 3 };
//    ^? { method: string; retries: number }        ← properties widen even in a const!
request(config.method);   // ❌ string is not assignable to Method
```

`const` prevents reassigning `config`, but the object's *properties* are mutable, so they widen. The fix:

```ts
const config = { method: "GET", retries: 3 } as const;
//    ^? { readonly method: "GET"; readonly retries: 3 }
request(config.method);   // ✅
```

**`as const` does two things:** makes every property `readonly`, and prevents widening (recursively, including nested objects and arrays). It's the single most useful assertion in the language and appears constantly in real code.

```ts
// The pattern you'll use over and over:
const ROLES = ["owner", "admin", "viewer"] as const;
type Role = typeof ROLES[number];        // "owner" | "admin" | "viewer"
//   ^ one source of truth: the runtime array AND the type
```

That last idiom is worth memorising. Without `as const`, `ROLES` is `string[]` and `typeof ROLES[number]` is just `string` — useless. With it, you get a runtime value you can iterate *and* a precise union type, derived from one declaration. It's the correct replacement for `enum` in most cases (§7).

---

## 4. Unions: set union

```ts
type Id = string | number;              // { all strings } ∪ { all numbers }
```

**The rule that trips people up: on a union, you may only access what's on *every* member.**

```ts
type Cat = { name: string; meow(): void };
type Dog = { name: string; bark(): void };
type Pet = Cat | Dog;

function f(p: Pet) {
  p.name;        // ✅ present on both
  p.meow();      // ❌ Property 'meow' does not exist on type 'Pet'
}
```

That's set logic, not a limitation: a `Pet` value might be a `Dog`, and dogs have no `meow`. You must **narrow** first ([Lesson 07](../02-type-system/07-narrowing-and-control-flow.md)).

```ts
if ("meow" in p) p.meow();     // ✅ narrowed to Cat
```

---

## 5. Intersections: set intersection

```ts
type WithId   = { id: string };
type WithName = { name: string };
type Both = WithId & WithName;    // { id: string; name: string }
```

**Here's the part that feels backwards and is the classic interview question:**

> For **object** types, `&` produces an object with *more* properties — which is a **smaller** set of values, because fewer objects satisfy more requirements.

More properties = more constraints = fewer inhabitants. `A & B` is a **subtype** of both `A` and `B`. That's consistent with set intersection; it just reads oddly because "more fields" feels bigger.

For **primitives**, the intersection is usually empty:
```ts
type Impossible = string & number;      // never — no value is both
type StillString = string & {};          // string (roughly) — a real trick, see below
type Weird = "a" & "b";                  // never
```

### Two practical uses

```ts
// 1. Composition — extending a props type
type ButtonProps = BaseProps & { variant: "primary" | "ghost" };

// 2. Branded types — the intersection of a primitive with a phantom marker
type UserId = string & { readonly __brand: "UserId" };
// No plain string satisfies this, so you can't pass a raw string where a UserId is required.
// Full treatment in Lesson 09.
```

### The intersection footgun
```ts
type A = { kind: "a"; value: string };
type B = { kind: "b"; value: number };
type AB = A & B;
//   ^? { kind: never; value: never }     ← every property is impossible
const x: AB = { kind: "a", value: "s" };  // ❌ unassignable — AB has NO inhabitants
```
You probably wanted `A | B`. **If an intersection produces `never` properties, you meant a union.** That's a genuinely useful diagnostic.

---

## 6. `any` vs `unknown` vs `never` vs `void`

The four-way distinction is asked in essentially every TypeScript interview. Learn them as sets.

| | Set | Assignable **to** it | Assignable **from** it | Use for |
|---|---|---|---|---|
| **`any`** | opts out of the system | everything | **everything** ⚠️ | escape hatch, last resort |
| **`unknown`** | every value | everything | **nothing** (must narrow) | **every external input** |
| **`never`** | no values | nothing | everything | impossible states, exhaustiveness |
| **`void`** | "return value unusable" | `undefined` (and anything, in callbacks) | ~nothing useful | functions with no meaningful return |

### `any` — the infection

```ts
const data: any = JSON.parse(raw);
const n: number = data.user.name;      // no error. n is actually a string.
n.toFixed(2);                           // 💥 at runtime
```

`any` doesn't just disable checking on itself — **it disables checking on everything downstream.** One `any` at a boundary silently untypes an entire call chain. This is why the ESLint `no-unsafe-*` rules matter more than banning the keyword: most `any` arrives *implicitly* from an untyped library, not from someone typing it.

### `unknown` — `any`'s safe replacement

```ts
const data: unknown = JSON.parse(raw);
data.user;                              // ❌ 'data' is of type 'unknown'

// You must prove the shape before using it:
if (typeof data === "object" && data !== null && "user" in data) {
  // now `data` is narrowed
}
```

**The rule: `unknown` at every boundary.** `JSON.parse`, `fetch().json()`, `localStorage.getItem`, `process.env`, URL params, WebSocket messages, `catch` variables. Then parse it once, deliberately ([Lesson 15](../04-real-code/15-the-boundary-and-parsing.md)).

```ts
// ❌ the lie
function handle(payload: any) { return payload.amount * 2; }

// ✅ the truth, with the check forced on you
function handle(payload: unknown) {
  const parsed = PaymentSchema.parse(payload);   // throws if wrong
  return parsed.amount * 2;
}
```

### `never` — the empty set, and why it's useful

`never` looks academic and is one of the most practically useful types in the language.

```ts
// 1. A function that never returns normally
function fail(msg: string): never { throw new Error(msg); }
function loop(): never { while (true) {} }

// 2. Exhaustiveness checking — THE killer use
type Status = "pending" | "succeeded" | "failed";

function label(s: Status): string {
  switch (s) {
    case "pending":   return "Pending";
    case "succeeded": return "Paid";
    case "failed":    return "Failed";
    default:
      const _exhaustive: never = s;   // ← if a case is missing, s isn't never → compile error
      throw new Error(`Unhandled: ${s}`);
  }
}
```
**Add `"refunded"` to `Status` and this file fails to compile**, pointing at the exact place you forgot. That's the type system doing real work — and it's the direct answer to the "adding an enum value is a breaking change" problem from [API Lesson 11](../../API/02-rest-design/11-versioning-and-evolution.md).

```ts
// 3. Impossible states
type Result<T> = { ok: true; value: T } | { ok: false; error: string };
// After `if (r.ok)`, the else branch has `value: never` — it can't exist.

// 4. Filtering unions (Lesson 10)
type NonNull<T> = T extends null | undefined ? never : T;
//                                             ^ removed from the union

// 5. never in a union disappears
type X = string | never;    // string  ← because never adds no values
```

### `void` vs `undefined` — a real distinction

```ts
function a(): void {}            // "don't use my return value"
function b(): undefined { return undefined; }   // "I return the value undefined"

const x: void = undefined;       // ✅
const y: undefined = undefined;  // ✅
```

The important difference is in **callback positions**, and it's deliberate:

```ts
type Handler = () => void;
const h: Handler = () => 42;     // ✅ allowed! The return value is just ignored.

// Which is why this works — and is genuinely useful:
[1, 2, 3].forEach(n => arr.push(n));    // push returns number; forEach wants void
```

But it also causes a real bug:
```ts
// ❌ The classic React useEffect mistake
useEffect(() => fetchData(), []);
//        ^ returns Promise<void>. React expects a CLEANUP FUNCTION or undefined.
//          A Promise isn't callable → React would try to call it on unmount.

useEffect(() => { void fetchData(); }, []);   // ✅ braces discard the return
```
You'll meet this again in [React Lesson 08](../../React/README.md). The `void` operator (`void expr`) is the idiomatic way to say "I'm deliberately discarding this."

---

## 6b. `null` vs `undefined` — pick a convention

```ts
let a: string | undefined;    // "not set yet" / absent
let b: string | null;         // "explicitly empty"
```

JavaScript has two nullish values, which is one too many. Conventions that work:
- **`undefined` for absence, `null` for explicit emptiness.** This matches JSON (`undefined` disappears in `JSON.stringify`, `null` survives) and matches PATCH semantics: absent = unchanged, `null` = clear ([API L09](../../API/02-rest-design/09-writes-patch-and-bulk.md)).
- **`undefined` only, internally** — simpler, and what most codebases converge on. Use `?.` and `??` throughout.

**What matters is picking one and being consistent.** And note `?? ` vs `||`:
```ts
const port = env.PORT ?? 3000;       // ✅ only null/undefined trigger the default
const port2 = env.PORT || 3000;      // ❌ "" and 0 also trigger it — a real bug source
```

---

## 7. `enum` vs union of literals vs `as const` — the decision

A frequent interview question because the answer is "usually not `enum`", which surprises people.

```ts
// 1. enum — the only construct here that EMITS RUNTIME CODE
enum Status { Pending, Succeeded }
// Compiles to a real object with reverse mappings. Also: Status.Pending === 0,
// so `if (status)` is false for Pending. That's a real bug source.

// 2. String enum — better, still emits code
enum Status2 { Pending = "pending", Succeeded = "succeeded" }

// 3. const enum — inlined, no runtime object
const enum Status3 { Pending = "pending" }
// ⚠️ Incompatible with isolatedModules/esbuild/Vite. Avoid.

// 4. Union of string literals — zero runtime cost  ← usually the answer
type Status4 = "pending" | "succeeded";

// 5. as const object — when you need to iterate at runtime  ← the other answer
const STATUS = { Pending: "pending", Succeeded: "succeeded" } as const;
type Status5 = typeof STATUS[keyof typeof STATUS];   // "pending" | "succeeded"
```

| | Runtime cost | Iterable | Literal-friendly | Verdict |
|---|---|---|---|---|
| numeric `enum` | object emitted | yes | no (numbers in JSON!) | ❌ avoid |
| string `enum` | object emitted | yes | must import the enum to pass a value | ⚠️ ok, rarely needed |
| `const enum` | none (inlined) | no | yes | ❌ breaks modern bundlers |
| **string literal union** | **none** | no | **yes** — pass `"pending"` directly | ✅ **default** |
| **`as const` object** | one frozen object | **yes** | yes | ✅ when you need the values at runtime |

**The two decisive arguments against `enum`:**
1. **It emits runtime code**, which makes it the only "type" that isn't erased — surprising, and it defeats tree-shaking.
2. **It's nominally typed**, uniquely in TypeScript. You cannot pass `"pending"` where `Status.Pending` is expected; you must import the enum. That's friction at every call site and every JSON boundary — and it means your API's string values and your internal enum can silently diverge.

> **The answer to give:** *"Union of string literals by default — zero runtime cost, and the values are just strings so they cross the JSON boundary cleanly. An `as const` object when I also need to iterate the values at runtime. `enum` only when a framework requires it, because it's the one TypeScript construct that isn't erased and the one place TypeScript is nominally typed."*

---

## 8. Arrays, tuples and `readonly`

```ts
let a: string[];                    // any length
let b: Array<string>;               // identical
let c: [string, number];            // TUPLE — exactly 2, in that order
let d: [string, ...number[]];       // at least 1, rest numbers
let e: [x: number, y: number];      // named tuple elements — better hovers
let f: readonly string[];           // no push/pop/splice
let g: ReadonlyArray<string>;       // identical
let h: readonly [string, number];   // readonly tuple
```

Tuples are what make `useState` work:
```ts
function useState<T>(init: T): [T, (v: T) => void] { /* ... */ }
const [count, setCount] = useState(0);   // count: number, setCount: (v: number) => void
```

**`readonly` is compile-time only** (it's erased — Lesson 01), but it's genuinely valuable as intent:
```ts
function sum(nums: readonly number[]) {
  nums.push(1);          // ❌ caught at compile time
  return nums.reduce((a, b) => a + b, 0);
}
// Accepting `readonly T[]` also means callers can pass a ReadonlyArray. Strictly more general.
```
**Rule: accept `readonly T[]` in function parameters** unless you genuinely mutate. It's a wider accepted set and documents intent. Note the direction: `T[]` is assignable to `readonly T[]`, but not vice versa.

For deep immutability at runtime you need `Object.freeze`, and for deep immutability in types you need a recursive mapped type ([Lesson 11](../03-type-level/11-mapped-and-template-literal-types.md)).

---

## 9. Production rules

| Rule | Why |
|---|---|
| **`unknown` at every boundary; never `any`** | `any` disables checking on everything downstream |
| **Union of string literals over `enum`** | Zero runtime cost, crosses JSON cleanly, structurally typed |
| **`as const` for config objects and value lists** | One source of truth for the runtime value *and* the type |
| **`typeof ARR[number]` to derive a union from an array** | Never write the same list twice |
| **An exhaustiveness `never` check in every switch over a union** | Adding a member becomes a compile error, not a silent gap |
| **`readonly T[]` in parameters unless you mutate** | Accepts more, documents intent |
| **Pick one of `null`/`undefined` and be consistent** | Two nullish values is one too many |
| **`??` not `||` for defaults** | `||` also fires on `""` and `0` |
| **Annotate the variable rather than asserting at the call site** | `let m: Method = "GET"` beats `"GET" as Method` |
| **If an intersection gives you `never` properties, you meant a union** | Reliable diagnostic |

---

## 10. Interview traps

**Q1. "`any` vs `unknown` vs `never` vs `void`?"**
As sets: `any` opts out (assignable both ways — dangerous); `unknown` is every value (assignable *from* everything, *to* nothing — so you must narrow); `never` is no values (assignable *to* everything, vacuously); `void` means the return value is unusable. Then the practical line: *"`unknown` at boundaries, `never` for exhaustiveness, `void` for callbacks, `any` as a last resort with a comment explaining why."*

**Q2. "Why is `never` assignable to everything?"**
Vacuous truth: the assignability rule is "every value in the source set is in the target set", and the empty set has no values to violate it. It's the bottom type.

**Q3. "Why does `const x = 'a'` give `'a'` but `let x = 'a'` give `string`?"**
Widening. `let` can be reassigned, so TypeScript widens to the base type to keep future assignments valid; `const` can't, so the literal type is kept. And **properties of a `const` object still widen** unless you use `as const` — that's the part that catches people.

**Q4. "What does `as const` do?"**
Prevents literal widening and makes everything `readonly`, recursively. Its highest-value use is `const X = [...] as const; type T = typeof X[number]` — one declaration giving you both a runtime array and a precise union.

**Q5. "Why does `A & B` have more properties but represent fewer values?"**
Because more required properties means fewer objects satisfy the type. It's genuinely set intersection; it only *reads* backwards. And if an intersection yields `never` properties, you wanted a union.

**Q6. "`enum` or union of string literals?"**
Union, by default. Two reasons: `enum` **emits runtime code** (the only non-erased construct) and it's **nominally typed** (you can't pass `"pending"` where `Status.Pending` is expected), which adds friction at every call site and every JSON boundary. `as const` object when you need to iterate the values.

**Q7. "How do you make a switch exhaustive?"**
`default: const _x: never = value;`. If a union member is unhandled, `value` isn't `never` and it's a compile error. Mention that this is what makes adding an enum value safe — the compiler shows you every place to update.

**Q8. "Why is `() => 42` assignable to `() => void`?"**
Because `void` in a return position means "the caller won't use the return value", not "the function returns nothing". It's deliberate, and it's what makes `arr.forEach(x => set.add(x))` work. The bug it enables is `useEffect(() => fetchData(), [])` — React receives a Promise where it expected a cleanup function.

**Q9. "`readonly` — is it enforced at runtime?"**
No. It's erased like everything else; it's a compile-time constraint only. `Object.freeze` is the runtime version. And accepting `readonly T[]` in a parameter is strictly more general than `T[]`.

**Q10. "What's the difference between `?? ` and `||`?"**
`??` only falls back on `null`/`undefined`; `||` falls back on any falsy value, so `""`, `0` and `false` trigger it. `port || 3000` when `PORT="0"` is a real bug.

---

## 11. Build & break

### Build — the assignability lab
Create `src/sets.ts`. For each line, **predict pass/fail before checking**, then hover to see the type.

```ts
type Small = "a" | "b";
type Big = "a" | "b" | "c";

let s: Small = null as any as Big;   // ?
let b: Big   = null as any as Small; // ?

let u: unknown = "hello";            // ?
let str: string = u;                 // ?
let n: never = u as never;           // ?
let anything: string = null as any as never;  // ?

type Obj = { a: string } & { b: number };
const o: Obj = { a: "x" };           // ?

type Bad = { k: "a" } & { k: "b" };
const bad: Bad = { k: "a" };         // ?  and hover Bad — what is `k`?

const arr = ["x", "y"];              // hover: ?
const arr2 = ["x", "y"] as const;    // hover: ?
type El = typeof arr2[number];       // hover: ?
```

### Build — the exhaustiveness pattern
```ts
export const PAYMENT_STATUSES = [
  "requires_payment_method", "requires_capture", "succeeded", "failed", "refunded",
] as const;
export type PaymentStatus = typeof PAYMENT_STATUSES[number];

export function statusLabel(s: PaymentStatus): string {
  switch (s) {
    case "requires_payment_method": return "Awaiting payment method";
    case "requires_capture":        return "Authorised";
    case "succeeded":               return "Paid";
    case "failed":                  return "Failed";
    case "refunded":                return "Refunded";
    default: {
      const _exhaustive: never = s;
      throw new Error(`Unhandled status: ${_exhaustive}`);
    }
  }
}
```
Now **add `"disputed"` to the array** and watch `statusLabel` fail to compile with a precise error. That's the mechanism that makes an API's open enum safe on the client — and it's the concrete answer to API Lesson 11's enum problem.

### Break — five experiments
1. **`any` infection.** `const d: any = JSON.parse("{}"); const n: number = d.a.b.c; n.toFixed(2);` — zero compile errors, runtime crash. Change `any` to `unknown` and count how many errors appear. That count is what `unknown` bought you.
2. **Widening.** `const cfg = { method: "GET" }` then pass `cfg.method` to a function taking `"GET" | "POST"`. Fails. Add `as const`. Passes.
3. **`never` intersection.** Write `type X = { k: "a" } & { k: "b" }` and hover `X["k"]`. It's `never`. Try to construct a value.
4. **The `void` callback bug.** `useEffect(() => fetchData(), [])` in a React file (or any function typed `() => void`). Note it compiles, then reason about what React receives.
5. **The `||` bug.** `const retries = opts.retries || 3` with `opts.retries = 0`. Watch `0` become `3`. Fix with `??`.

### Explain out loud (90 seconds)
1. Types as sets, and what "assignable" means.
2. Where `any`, `unknown` and `never` sit in the hierarchy, and why `never` is assignable to everything.
3. Why `let` widens and `const` doesn't, and what `as const` fixes.
4. Why intersections of object types have more properties but fewer values.
5. Why you'd choose a string literal union over an `enum`.

---

## What's next

You have the mental model. Next: object types in detail — `interface` vs `type` (a real question with real answers), excess property checks (the rule that confuses everyone once), optional vs `undefined`, index signatures, and how to model shapes precisely.

Next → **[Lesson 04: Objects, interfaces vs types, and the shape rules](04-objects-and-interfaces.md)**
