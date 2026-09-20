# Lesson 07 — Narrowing & control-flow analysis

> **Why this lesson exists:** narrowing is where TypeScript stops being a nuisance and starts being a collaborator. It reads your `if` statements and *proves* things about your values. But it has precise limits — and every *"why isn't this narrowed?!"* frustration is one of about six specific limits. Learn them and the compiler starts feeling like it's on your side.

**Time:** ~70 minutes · **Prereq:** Lesson 06

---

## 1. The idea in one sentence

> **TypeScript simulates your control flow and tracks, at every point in the program, the narrowest type each variable could have — and the discriminated union is the pattern that makes this power available to your own domain types.**

---

## 2. The seven narrowing mechanisms

### 1. `typeof`
```ts
function f(x: string | number | null) {
  if (typeof x === "string") x.toUpperCase();       // string
  else if (typeof x === "number") x.toFixed(2);     // number
  else x;                                            // null
}
```
**The trap:** `typeof null === "object"`.
```ts
function g(x: string | null) {
  if (typeof x === "object") x;      // null   ← catches null, not an object!
}
```

### 2. Truthiness
```ts
function f(x: string | null | undefined) {
  if (x) x.toUpperCase();            // string  (narrowed to non-empty, but typed string)
  else x;                             // string | null | undefined  ← "" is still a string!
}
```
**The trap that causes real bugs:**
```ts
function render(count: number | undefined) {
  if (!count) return "none";         // ⚠️ 0 takes this branch
  return `${count} items`;
}
render(0);      // "none" — probably wrong

// ✅ be explicit about what you're checking
if (count === undefined) return "none";
if (count == null) return "none";         // covers null AND undefined (the one OK use of ==)
```
Same class of bug as `||` vs `??` (Lesson 03). **With `strictNullChecks`, prefer explicit `=== undefined` / `== null` over truthiness for numbers and strings.**

### 3. Equality
```ts
function f(a: string | number, b: string | boolean) {
  if (a === b) { a; b; }             // both narrowed to string — the only overlap!
}

function g(x: "a" | "b" | "c") {
  if (x !== "a") x;                  // "b" | "c"
}
```
Narrowing by *inequality* on literal unions is how you eliminate members one at a time.

### 4. The `in` operator
```ts
type Cat = { name: string; meow(): void };
type Dog = { name: string; bark(): void };

function speak(pet: Cat | Dog) {
  if ("meow" in pet) pet.meow();     // Cat
  else pet.bark();                    // Dog
}
```
Useful when there's no explicit discriminant — but a real discriminant is better (§3).

### 5. `instanceof`
```ts
function f(x: Date | string) {
  if (x instanceof Date) x.toISOString();
  else x.toUpperCase();
}

// Works on classes only. Also the correct way to handle catch (Lesson 02):
try { risky(); } catch (e) {
  if (e instanceof ApiError) console.log(e.status);
  else if (e instanceof Error) console.log(e.message);
  else console.log(String(e));       // JS can throw anything
}
```
**The trap:** `instanceof` fails across realms (iframes, Node's `vm`, some bundling setups produce two copies of a class). `Array.isArray(x)` exists precisely because `x instanceof Array` is unreliable — a nice detail to know.

### 6. Type predicates — narrowing you write yourself
```ts
function isPayment(x: unknown): x is Payment {
  return typeof x === "object" && x !== null
    && "id" in x && typeof (x as any).id === "string"
    && "amountMinor" in x && typeof (x as any).amountMinor === "number";
}

const data: unknown = await res.json();
if (isPayment(data)) data.amountMinor;    // ✅ narrowed
```

**Their killer use is `Array.filter`:**
```ts
const maybe: (string | null)[] = ["a", null, "b"];

const bad = maybe.filter(x => x !== null);
//    ^? (string | null)[]   ← filter can't know. Still nullable!

const good = maybe.filter((x): x is string => x !== null);
//    ^? string[]            ✅

// A reusable helper you'll want in every project:
export function isNotNullish<T>(x: T): x is NonNullable<T> { return x != null; }
const clean = maybe.filter(isNotNullish);   // string[]
```
> TypeScript 5.5 added **inferred type predicates**, so `maybe.filter(x => x !== null)` now narrows correctly in many simple cases. The explicit helper still works everywhere and is clearer for non-trivial conditions — and knowing *why* it was needed is the point.

**The danger:** the compiler trusts a predicate absolutely.
```ts
function isPayment(x: unknown): x is Payment { return true; }   // 🔥 a silent lie
```
A wrong predicate is exactly as unsafe as `as`. Which is why schema validation ([Lesson 15](../04-real-code/15-the-boundary-and-parsing.md)) beats hand-written predicates — the validator and the type come from one source.

### 7. Assertion functions
```ts
function assert(cond: unknown, msg?: string): asserts cond {
  if (!cond) throw new Error(msg ?? "Assertion failed");
}
function assertIsPayment(x: unknown): asserts x is Payment {
  if (!isPayment(x)) throw new Error("not a payment");
}

const p: unknown = load();
assertIsPayment(p);
p.amountMinor;              // ✅ narrowed for the rest of the scope

const maybe: string | undefined = get();
assert(maybe, "required");
maybe.toUpperCase();        // ✅
```
Predicate = returns a boolean to branch on. Assertion = throws, narrowing everything after it. Assertions **require an explicit annotation** (you can't infer `asserts`).

---

## 3. Discriminated unions — the single most valuable pattern

A union where every member has a common **literal** property that identifies it.

```ts
type Result<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string; retryable: boolean };

function render(r: Result<Payment>) {
  switch (r.status) {
    case "loading": return <Spinner />;
    case "success": return <View payment={r.data} />;      // ✅ data exists here
    case "error":   return <Err msg={r.error} retry={r.retryable} />;
  }
}
```

**Why this is the pattern to reach for**, and it's worth being able to argue:

```ts
// ❌ The optional-field soup that most codebases have
type BadResult<T> = {
  loading: boolean;
  data?: T;
  error?: string;
};
// This type permits 8 combinations, of which 3 are meaningful.
// { loading: true, data: x, error: "e" } is representable and nonsensical.
// Every consumer must write defensive checks the compiler can't verify.

// ✅ The discriminated union permits exactly 3 states, and each carries
//    exactly the data it needs — no more, no less.
```

**Illegal states become unrepresentable**, not merely unlikely. This is the central idea of [Lesson 09](../03-type-level/09-illegal-states-unrepresentable.md), and it maps directly onto the state machines from [API Lesson 09](../../API/02-rest-design/09-writes-patch-and-bulk.md) — the API enforces transitions in SQL; the client enforces state shapes in types.

### Rules for good discriminants
```ts
// ✅ literal types — string is the most readable
type A = { kind: "a" } | { kind: "b" };
// ✅ booleans work — good for two-state results
type R<T> = { ok: true; value: T } | { ok: false; error: Error };
// ❌ non-literal types can't discriminate
type Bad = { kind: string } | { kind: number };      // no narrowing possible
```

The discriminant must be a **literal type** on every member, and it must be the same property name. Then `switch`, `if`, ternaries and destructuring all narrow.

```ts
// Destructuring a discriminated union works — but only when destructured
// AFTER narrowing, or via the discriminant itself:
function f(r: Result<Payment>) {
  const { status } = r;
  if (status === "success") r.data;      // ✅ TS 4.6+ correlates them
}
```

### Exhaustiveness — making additions safe
```ts
function label(r: Result<Payment>): string {
  switch (r.status) {
    case "loading": return "Loading…";
    case "success": return "Done";
    case "error":   return "Failed";
    default: return assertNever(r);      // ← add a state, THIS breaks
  }
}
export function assertNever(x: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(x)}`);
}
```
**This is the mechanism that makes an open API enum safe on the client.** Add `"refunded"` to a status union and every non-exhaustive switch fails to compile, pointing at exactly what to update.

---

## 4. The six limits of narrowing (why it "stops working")

Every narrowing frustration is one of these. Knowing them by name is the difference between fighting the compiler and working with it.

### Limit 1 — Narrowing resets inside callbacks
```ts
function f(x: string | undefined) {
  if (x) {
    setTimeout(() => x.toUpperCase(), 0);       // ❌ 'x' is possibly 'undefined'
  }
}
```
**Why:** the callback runs later, and `x` might have been reassigned in between. TypeScript can't prove it hasn't.

```ts
// Fix 1 — copy to a const (the compiler knows a const can't change)
if (x) { const s = x; setTimeout(() => s.toUpperCase(), 0); }

// Fix 2 — make the parameter const-like. Narrowing IS preserved for
// parameters/consts that are never reassigned in the function:
function f2(x: string | undefined) {
  if (!x) return;
  setTimeout(() => x.toUpperCase(), 0);        // ✅ x is never reassigned → preserved
}
```
That second fix surprises people: **TypeScript preserves narrowing into closures when the variable is a `const` or a never-reassigned parameter.** It's only `let`/`var` that reset.

### Limit 2 — Narrowing doesn't survive a function call
```ts
function f(x: { a?: string }) {
  if (x.a) {
    doSomething();
    x.a.toUpperCase();       // ✅ actually fine for properties of a param...
  }
}

let obj: { a?: string } = get();
if (obj.a) {
  mutate();                  // could this have changed obj.a?
  obj.a.toUpperCase();       // ✅ TS assumes not — this is UNSOUND but pragmatic
}
```
TypeScript **assumes function calls don't mutate your objects.** That's unsound (a called function could clear the property) but the alternative — invalidating all narrowing after every call — would make the language unusable. Worth knowing as one more deliberate hole.

### Limit 3 — Property narrowing doesn't correlate across objects
```ts
type Payment = { status: "succeeded"; capturedAt: string } | { status: "failed"; code: string };

function f(a: Payment, b: Payment) {
  if (a.status === "succeeded") {
    a.capturedAt;      // ✅
    b.capturedAt;      // ❌ b is independent
  }
}
```
Obvious when stated; a source of confusion when you have `props.payment` and `payment` aliases.

### Limit 4 — `let` reassignment widens back
```ts
let x: string | number = "a";
if (typeof x === "string") {
  x = 5;                     // legal — x's declared type allows it
  x.toUpperCase();           // ❌ x is now number
}
```
Control flow analysis tracks assignments, which is correct and occasionally surprising.

### Limit 5 — Generics don't narrow the way you expect
```ts
function f<T extends string | number>(x: T) {
  if (typeof x === "string") {
    x.toUpperCase();         // ✅ narrows to T & string
    const y: T = x;          // ❌ T & string isn't assignable back to T
  }
}
```
Narrowing a type *parameter* gives you an intersection, not the branch type. This trips people constantly ([Lesson 08](08-generics-properly.md)).

### Limit 6 — Index and dynamic access don't narrow
```ts
const key = "a" as "a" | "b";
const obj = { a: "x", b: 2 };
if (typeof obj[key] === "string") {
  obj[key].toUpperCase();    // ❌ no narrowing through a dynamic key
}
```
Fix: extract to a variable first — `const v = obj[key]; if (typeof v === "string") v.toUpperCase();`

### Aliased conditions — the good news (TS 4.4+)
```ts
function f(x: string | number) {
  const isString = typeof x === "string";     // must be a const
  if (isString) x.toUpperCase();               // ✅ works!
}
```
TypeScript tracks conditions stored in `const`s. Works for discriminant checks too:
```ts
const isSuccess = r.status === "success";
if (isSuccess) r.data;                         // ✅
```
Requires `const` (not `let`) and a directly-analysable expression.

---

## 5. `!`, `?.` and `??`

```ts
const el = document.getElementById("x")!;      // non-null assertion
```
**`!` is an assertion, with all the risks of `as`.** It says "trust me, not null." If you're wrong, you get a runtime error and the compiler was told not to help.

```ts
// ❌ the lazy version
const el = document.getElementById("root")!;
el.append(node);                    // 💥 if #root doesn't exist

// ✅ handle it
const el = document.getElementById("root");
if (!el) throw new Error("#root not found — check index.html");
el.append(node);

// ✅ or an assertion function that actually checks
assert(el, "#root not found");
el.append(node);
```
**When `!` is genuinely fine:** after a check the compiler can't see (a `Map` you just `set`), in tests, and in code where you've established the invariant three lines up. **Always the exception, never the habit** — and a `// !` with no comment in a code review is a legitimate finding.

```ts
// Optional chaining and nullish coalescing — use these instead
const name = user?.profile?.name ?? "Anonymous";
user?.save?.();                     // call only if it exists
arr?.[0];                            // index only if arr exists

// The precedence trap:
const x = a ?? b || c;              // ❌ SyntaxError — must parenthesise
const y = (a ?? b) || c;            // ✅
```

---

## 6. Ledger's narrowing patterns

```ts
// A discriminated union for every async state — one type, used everywhere
export type Async<T, E = ApiError> =
  | { state: "idle" }
  | { state: "loading" }
  | { state: "success"; data: T }
  | { state: "error"; error: E };

// The state machine, mirroring the API (API Lesson 09)
export type Payment =
  | { status: "requires_payment_method"; id: PaymentId; money: Money }
  | { status: "requires_capture"; id: PaymentId; money: Money; authorizedAt: string }
  | { status: "succeeded"; id: PaymentId; money: Money; capturedAt: string }
  | { status: "failed"; id: PaymentId; money: Money; failureCode: FailureCode }
  | { status: "refunded"; id: PaymentId; money: Money; refundedMoney: Money };

// Actions the compiler can prove are legal
export function canCapture(p: Payment): p is Extract<Payment, { status: "requires_capture" }> {
  return p.status === "requires_capture";
}
export function canRefund(p: Payment): p is Extract<Payment, { status: "succeeded" }> {
  return p.status === "succeeded";
}

// Now the UI can't offer an illegal action:
function Actions({ payment }: { payment: Payment }) {
  return (
    <>
      {canCapture(payment) && <CaptureButton authorizedAt={payment.authorizedAt} />}
      {canRefund(payment)  && <RefundButton  capturedAt={payment.capturedAt} />}
    </>
  );
}
// `payment.capturedAt` is only *readable* inside canRefund's branch. You cannot
// render a refund button for a failed payment — the data isn't in the type.
```

That's the payoff: **the UI's legal actions and the API's legal transitions are enforced by the same shape.** A capture timestamp is unreachable on a failed payment because it isn't in that variant. That's stronger than validation — it's unrepresentable.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Discriminated unions over optional-field soup** | 3 valid states instead of 8 representable combinations |
| **A literal discriminant on every union you own** | `string`/`number` discriminants can't narrow |
| **`assertNever` in every `default` branch** | Adding a variant becomes a compile error |
| **Explicit `=== undefined` / `== null` over truthiness for numbers and strings** | `0` and `""` are falsy and valid |
| **Type predicates for `.filter()`** | Otherwise `filter(x => x !== null)` keeps `null` in the type (pre-5.5) |
| **Copy to a `const` before using in a callback** — or don't reassign the variable | Narrowing resets for reassignable bindings |
| **`instanceof` for `catch`, and handle the non-`Error` case** | JavaScript can throw anything |
| **`Array.isArray` not `instanceof Array`** | Cross-realm reliability |
| **Avoid `!`; if you use one, comment the invariant** | It's an assertion with all of `as`'s risks |
| **Never write a type predicate that doesn't fully check** | The compiler trusts it absolutely |
| **Prefer schema validation over hand-written predicates at boundaries** | One source of truth for the check and the type |

---

## 8. Interview traps

**Q1. "What is narrowing?"**
Control-flow analysis: TypeScript simulates your code paths and tracks the narrowest possible type of each variable at each point. Mechanisms: `typeof`, truthiness, equality, `in`, `instanceof`, type predicates, assertion functions, and discriminant checks.

**Q2. "What's a discriminated union and why does it matter?"**
A union whose members share a literal-typed discriminant property, enabling exhaustive narrowing. It matters because it makes illegal states **unrepresentable** — the optional-field alternative (`{loading, data?, error?}`) permits 8 combinations of which 3 are meaningful, and the compiler can't help you with the other 5.

**Q3. "How do you make a switch exhaustive?"**
`default: return assertNever(x)` where `assertNever(x: never): never`. If a variant is unhandled, `x` isn't `never` and it's a compile error. **This is how adding an API enum value becomes safe** — every non-exhaustive switch breaks and shows you where.

**Q4. "Why doesn't this narrow inside my callback?"**
```ts
if (x) setTimeout(() => x.toUpperCase(), 0);
```
Because the callback runs later and a reassignable `x` might have changed. Fix: copy to a `const`, or **don't reassign** — TypeScript preserves narrowing into closures for `const`s and never-reassigned parameters. That second half is the part most people don't know.

**Q5. "What's wrong with `if (!count)` where `count: number | undefined`?"**
`0` is falsy, so a legitimate zero takes the "missing" branch. Use `count === undefined`. Same family as `||` vs `??`.

**Q6. "Type predicate vs assertion function vs `as`?"**
Predicate returns a boolean to branch on. Assertion throws and narrows for the rest of the scope. `as` just overrides the compiler with no runtime check. **All three are trusted unconditionally**, so a wrong predicate is as dangerous as `as` — which is the argument for schema validation instead.

**Q7. "Why does `filter(x => x !== null)` still give a nullable type?"**
`filter`'s signature returns `T[]` — it has no way to know your callback narrows (pre-5.5). Fix with an explicit predicate: `filter((x): x is string => x !== null)`, or a reusable `isNotNullish`. TS 5.5 infers predicates for simple cases, but the explicit form always works.

**Q8. "When is `!` acceptable?"**
After an invariant the compiler can't see — a `Map.get` right after a `set`, a known-present DOM node in a test. Never as a way to silence an error you don't understand, and always with a comment naming the invariant.

**Q9. "Why doesn't `typeof x === 'string'` narrow a generic `T` to `string`?"**
Narrowing a type *parameter* produces `T & string`, not `string` — because `T` could be a subtype with extra structure. So you can use string methods but can't assign back to `T`. Design around it by narrowing the *argument* rather than the parameter.

**Q10. "Why is `typeof null === 'object'` a problem?"**
It means `if (typeof x === "object")` catches `null` — so a check meant to find objects finds `null` too. Always `x !== null && typeof x === "object"`, in that order.

---

## 9. Build & break

### Build — the async state union
`src/types/async.ts`, and then use it everywhere:
```ts
export type Async<T, E = ApiError> =
  | { state: "idle" }
  | { state: "loading" }
  | { state: "success"; data: T }
  | { state: "error"; error: E };

export const idle = (): Async<never> => ({ state: "idle" });
export const loading = (): Async<never> => ({ state: "loading" });
export const success = <T,>(data: T): Async<T> => ({ state: "success", data });
export const failure = <E,>(error: E): Async<never, E> => ({ state: "error", error });

/** Exhaustive fold — the compiler forces you to handle every state. */
export function match<T, E, R>(a: Async<T, E>, handlers: {
  idle: () => R; loading: () => R; success: (d: T) => R; error: (e: E) => R;
}): R {
  switch (a.state) {
    case "idle":    return handlers.idle();
    case "loading": return handlers.loading();
    case "success": return handlers.success(a.data);
    case "error":   return handlers.error(a.error);
    default:        return assertNever(a);
  }
}
```
Then **add a `"refetching"` state** and watch every `match` call fail to compile until you handle it. That's the mechanism working.

### Build — a validating predicate you can trust
```ts
export function isPayment(x: unknown): x is Payment {
  if (typeof x !== "object" || x === null) return false;
  const o = x as Record<string, unknown>;
  if (typeof o.id !== "string" || !o.id.startsWith("pi_")) return false;
  if (typeof o.status !== "string") return false;

  switch (o.status) {
    case "requires_payment_method": return true;
    case "requires_capture":        return typeof o.authorizedAt === "string";
    case "succeeded":               return typeof o.capturedAt === "string";
    case "failed":                  return typeof o.failureCode === "string";
    default:                        return false;    // unknown status → reject
  }
}
```
Note it validates **per variant** — the whole point of the discriminated union is that each variant has different required fields, so a predicate that only checks `id` is a lie. Then compare this to the Zod version in [Lesson 15](../04-real-code/15-the-boundary-and-parsing.md) and note how much less code says the same thing, with the type *derived* rather than duplicated.

### Break — six experiments
1. **The falsy-zero bug.** `function f(n: number | undefined) { if (!n) return "none"; }` — call with `0`. Then fix with `=== undefined`.
2. **The closure reset.** Reproduce the `setTimeout` narrowing error with a `let` and then with a never-reassigned parameter. Note that only one errors.
3. **The lying predicate.** `function isUser(x: unknown): x is User { return true; }` then `isUser("hi") && x.email.toUpperCase()`. Compiles, crashes.
4. **`typeof null`.** `if (typeof x === "object") x.foo` with `x: {foo: string} | null`. Read the error.
5. **Optional-field soup.** Model a loading state as `{loading: boolean, data?: T, error?: string}`, then write down all 8 representable combinations and mark which are meaningful. Convert to a discriminated union and count again.
6. **Non-exhaustive switch.** Remove `assertNever` from a switch, add a union member, and confirm the code compiles with a silently missing case. Put it back.

### Explain out loud (90 seconds)
1. The seven narrowing mechanisms.
2. Why a discriminated union beats optional fields — with the 8-vs-3 argument.
3. Three limits of narrowing, and the fix for each.
4. Why a type predicate is as dangerous as `as`.
5. How exhaustiveness checking makes an API enum addition safe.

---

## What's next

Narrowing lets you go from wide to precise. **Generics** let you write code that works across types while *preserving* precision — and they're where most TypeScript confusion lives, because inference has rules people have never been taught. Next: those rules.

Next → **[Lesson 08: Generics, properly](08-generics-properly.md)**
