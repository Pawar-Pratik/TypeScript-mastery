# Lesson 09 — Making illegal states unrepresentable

> **Why this lesson exists:** this is the highest-value idea in typed programming, and it's a *design* skill rather than a syntax one. Most TypeScript is used defensively — catch the wrong value after someone writes it. The shift is to model your domain so the wrong value **cannot be written down at all**. When you get this right, entire categories of bug stop existing and your tests get shorter.

**Time:** ~75 minutes · **Prereq:** Modules 1–2

---

## 1. The idea in one sentence

> **Design your types so the set of representable values equals the set of valid values — then "validation" happens once, at construction, and everything downstream is provably correct.**

The measurement that makes this concrete: **count the states your type permits, and count the states that are meaningful.** If those numbers differ, the gap is where your bugs live.

---

## 2. The counting exercise

```ts
// The type almost every codebase has:
type RequestState<T> = {
  isLoading: boolean;
  data?: T;
  error?: string;
};
```

**Representable states:** 2 (loading) × 2 (data present?) × 2 (error present?) = **8**.

**Meaningful states:** 3 — loading, loaded with data, failed with an error.

So **five of eight states are nonsense**, and every one is constructible:

| State | Meaning |
|---|---|
| `{ isLoading: true, data: x, error: "e" }` | Loading, but also succeeded and failed? |
| `{ isLoading: false }` | Not loading, no data, no error — did anything happen? |
| `{ isLoading: false, data: x, error: "e" }` | Both succeeded and failed |
| `{ isLoading: true, data: x }` | Stale data while loading (sometimes intended — see §6) |
| `{ isLoading: true, error: "e" }` | Retrying after an error (also sometimes intended) |

And the consequence in every consumer:
```ts
// Defensive code the compiler cannot verify — and which silently disagrees between files
if (state.error) return <Error/>;
if (state.isLoading) return <Spinner/>;
if (state.data) return <View data={state.data}/>;
return null;                      // ← this branch exists because the type allows it
```
That final `return null` is the tell. **A `return null` that "can't happen" means your type permits a state your logic doesn't.**

### The fix

```ts
type RequestState<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: ApiError };
```

**Representable states: 3. Meaningful states: 3.** The gap is zero.

```ts
switch (state.status) {
  case "loading": return <Spinner/>;
  case "success": return <View data={state.data}/>;    // data GUARANTEED here
  case "error":   return <Error e={state.error}/>;     // error GUARANTEED here
  default:        return assertNever(state);            // and nothing else exists
}
```

No `return null`. No defensive checks. `data` is not optional inside `success` — **it's not optional, it's present**, and the compiler knows.

> **The interview framing:** *"I count representable states versus meaningful states. If a type permits eight combinations and three are meaningful, the other five are bugs waiting to be constructed, and every consumer has to write defensive code the compiler can't verify. A discriminated union closes the gap, and then the exhaustiveness check makes adding a state safe."*

---

## 3. Technique 1 — Discriminated unions for domain states

The Ledger payment, modelled so that **each state carries exactly its own data**:

```ts
// ❌ the flat version — every field optional, nothing guaranteed
type Payment = {
  id: string;
  status: "requires_capture" | "succeeded" | "failed" | "refunded";
  authorizedAt?: string;
  capturedAt?: string;
  failureCode?: string;
  refundedMoney?: Money;
};
// `payment.capturedAt!` appears in the codebase within a week.

// ✅ each variant carries what it needs, and nothing it doesn't
type Payment =
  | { status: "requires_payment_method"; id: PaymentId; money: Money }
  | { status: "requires_capture";        id: PaymentId; money: Money; authorizedAt: IsoDate }
  | { status: "succeeded";               id: PaymentId; money: Money; capturedAt: IsoDate }
  | { status: "failed";                  id: PaymentId; money: Money; failureCode: FailureCode }
  | { status: "refunded";                id: PaymentId; money: Money; capturedAt: IsoDate;
                                          refunded: Money; refundedAt: IsoDate };
```

What this buys, concretely:
- `capturedAt` is **unreachable** on a failed payment. Not "undefined" — *not in the type*.
- There is no `!` anywhere, because nothing is optional-but-actually-required.
- A function taking `Extract<Payment, { status: "succeeded" }>` can only be called with a succeeded payment.

```ts
// Operations that state their own precondition, in the type
type Capturable = Extract<Payment, { status: "requires_capture" }>;
type Refundable = Extract<Payment, { status: "succeeded" }>;

function capture(p: Capturable): Payment { /* ... */ }
function refund(p: Refundable, amount: Money): Payment { /* ... */ }

declare const p: Payment;
capture(p);                            // ❌ not assignable — good
if (p.status === "requires_capture") capture(p);   // ✅
```

**This is the client-side mirror of the API's state machine** ([API Lesson 09](../../API/02-rest-design/09-writes-patch-and-bulk.md)): the server enforces transitions in a `WHERE` clause, the client enforces them in the type system. Same invariant, two layers, neither trusting the other.

---

## 4. Technique 2 — Branded types (nominal typing on demand)

Structural typing means every ID is just a `string` ([Lesson 06](../02-type-system/06-structural-typing-and-variance.md)). Branding fixes that with zero runtime cost.

```ts
declare const __brand: unique symbol;
export type Brand<T, B extends string> = T & { readonly [__brand]: B };
```

Why a `unique symbol` rather than a plain `__brand: "UserId"` property: the symbol isn't nameable outside this module, so nobody can construct a branded value by writing the property literally. It's a stronger fence for the same runtime cost (zero).

### IDs
```ts
export type MerchantId = Brand<string, "MerchantId">;
export type PaymentId  = Brand<string, "PaymentId">;
export type CustomerId = Brand<string, "CustomerId">;

// ONE place where the assertion lives — inside a validating constructor
export function paymentId(s: string): PaymentId {
  if (!/^pi_[A-Za-z0-9]{10,}$/.test(s)) throw new Error(`Invalid payment id: ${s}`);
  return s as PaymentId;
}

declare function getPayment(id: PaymentId): Promise<Payment>;
declare const cid: CustomerId;

getPayment(cid);            // ❌ CustomerId is not PaymentId
getPayment("pi_3Nx8");      // ❌ a raw string is not a PaymentId
getPayment(paymentId("pi_3Nx8"));   // ✅ validated at the door
```

**Note the design:** `as PaymentId` appears exactly once per brand, inside a function that actually validates. That's the correct use of an assertion — a single audited place where you take responsibility, rather than sprinkled at call sites.

### Units and validated strings — where branding pays even more
```ts
export type Cents      = Brand<number, "Cents">;
export type Seconds    = Brand<number, "Seconds">;
export type Millis     = Brand<number, "Millis">;
export type IsoDate    = Brand<string, "IsoDate">;
export type Email      = Brand<string, "Email">;
export type SafeHtml   = Brand<string, "SafeHtml">;   // survived sanitisation

function setTimeout2(fn: () => void, ms: Millis) {}
const timeout = 30 as Seconds;
setTimeout2(fn, timeout);        // ❌ — a genuine 1000× bug, caught at compile time
```

The seconds/milliseconds confusion is a famously real bug class. So is "this string was sanitised" vs "this string came from a user" — `SafeHtml` means a function taking `SafeHtml` literally cannot be handed raw input.

### Money — the one that matters most for Ledger
```ts
export type Currency = "usd" | "eur" | "gbp" | "inr" | "jpy";

/** Money carries its currency IN THE TYPE, so cross-currency arithmetic can't compile. */
export type Money<C extends Currency = Currency> = {
  readonly amountMinor: number;
  readonly currency: C;
  readonly __money: unique symbol;   // prevents structural forgery
} extends infer M ? Omit<M, "__money"> & { readonly __money?: undefined } : never;

// Simpler, and good enough in practice:
export type Money2<C extends Currency = Currency> = {
  readonly amountMinor: number;
  readonly currency: C;
};

export function money<C extends Currency>(amountMinor: number, currency: C): Money2<C> {
  if (!Number.isSafeInteger(amountMinor)) throw new Error("amountMinor must be a safe integer");
  return { amountMinor, currency };
}

/** Same-currency addition is enforced by the type parameter. */
export function add<C extends Currency>(a: Money2<C>, b: Money2<C>): Money2<C> {
  return money(a.amountMinor + b.amountMinor, a.currency);
}

const usd = money(4999, "usd");
const eur = money(4999, "eur");
add(usd, usd);        // ✅
add(usd, eur);        // ❌ Types of property 'currency' are incompatible
```

**That last line is the lesson.** Adding USD to EUR is a financial correctness bug that no amount of testing reliably catches, and here it's a compile error. This is the type-system counterpart of [API Lesson 05](../../API/01-foundations/05-data-formats.md)'s money rules.

---

## 5. Technique 3 — `Result` instead of exceptions

```ts
export type Result<T, E = ApiError> =
  | { ok: true;  value: T }
  | { ok: false; error: E };

export const Ok  = <T,>(value: T): Result<T, never> => ({ ok: true, value });
export const Err = <E,>(error: E): Result<never, E> => ({ ok: false, error });
```

**Why:** a thrown exception is invisible in the type system. `function charge(): Promise<Payment>` says nothing about failure, so the compiler can't make you handle it. `Promise<Result<Payment, ChargeError>>` says exactly what can go wrong, and you can't read `value` without checking `ok`.

```ts
type ChargeError =
  | { code: "card_declined"; declineCode: DeclineCode; retryable: false }
  | { code: "insufficient_funds"; retryable: false }
  | { code: "processor_timeout"; retryable: true }
  | { code: "rate_limited"; retryAfterSec: number; retryable: true };

async function charge(input: ChargeInput): Promise<Result<Payment, ChargeError>> { /* ... */ }

const r = await charge(input);
r.value;                     // ❌ not on the union
if (r.ok) r.value;           // ✅
else {
  if (r.error.retryable) scheduleRetry(r.error);      // ✅ typed per variant
  else showDecline(r.error);
}
```

Note `retryable` on the error union — that's the `retryable` column from [API Lesson 10](../../API/02-rest-design/10-errors-and-problem-details.md)'s error table, encoded so the client *cannot* forget to branch on it.

### Combinators worth having
```ts
export function map<T, U, E>(r: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return r.ok ? Ok(fn(r.value)) : r;
}
export function flatMap<T, U, E>(r: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return r.ok ? fn(r.value) : r;
}
export function unwrapOr<T, E>(r: Result<T, E>, fallback: T): T {
  return r.ok ? r.value : fallback;
}
/** Wrap a throwing function at the boundary. */
export function tryCatch<T>(fn: () => T): Result<T, unknown> {
  try { return Ok(fn()); } catch (e) { return Err(e); }
}
```

### Be honest about when *not* to use it
| Use `Result` | Use exceptions |
|---|---|
| **Expected** failures the caller must handle: validation, declines, not-found, rate limits | **Unexpected** failures: programmer bugs, invariant violations, OOM |
| Anywhere you want the compiler to force a branch | Anywhere the right response is "crash and log" |
| Library/domain boundaries | Deep internal helpers where plumbing `Result` adds noise |

**Do not `Result`-ify everything.** Threading `Result` through twelve layers is worse than a `try/catch` at the boundary. The rule: *"`Result` for failures that are part of the domain; exceptions for failures that mean the program is wrong."* Saying that shows judgement rather than dogma.

> React's `useActionState`, TanStack Query's `{ data, error, status }`, and Rust's `Result` are all the same shape. You'll meet it again in [React Lesson 17](../../React/README.md).

---

## 6. Technique 4 — Modelling the states you actually have

The one honest complaint about strict discriminated unions: sometimes you *want* "stale data while refetching."

**Don't reach back for optional fields.** Model it:

```ts
type Query<T, E = ApiError> =
  | { status: "idle" }
  | { status: "loading" }                                       // first load, nothing yet
  | { status: "success"; data: T; isStale: false }
  | { status: "refetching"; data: T }                            // ← stale data + loading
  | { status: "error"; error: E }
  | { status: "error-with-stale"; error: E; data: T };           // ← failed refetch

// 6 states, all meaningful. `data` is present exactly where it exists.
```

That's still zero gap between representable and meaningful — you just discovered you had six states, not three. **Finding the extra states is the design work**, and it's usually a sign you understand the domain better than the original three-state model did.

### Other states worth naming
```ts
// Form fields — "untouched" is genuinely different from "valid"
type Field<T> =
  | { state: "pristine"; value: T }
  | { state: "editing"; value: T }
  | { state: "invalid"; value: T; errors: readonly string[] }
  | { state: "valid"; value: T };

// Optimistic updates (React Lesson 18) — the pending value is its own state
type Item<T> =
  | { state: "saved"; value: T }
  | { state: "pending"; value: T; previous: T }     // rollback target is in the type
  | { state: "failed"; value: T; previous: T; error: ApiError };
```
That `previous` field is the rollback target — **by putting it in the type, you make "optimistic update with no way to roll back" unrepresentable.**

---

## 7. Technique 5 — Making it impossible to *construct* a bad value

Types describe shapes; they don't run code. So combine them with a **smart constructor** — one function that validates and is the only way to produce the branded type.

```ts
export type Email = Brand<string, "Email">;

// Private-by-convention: not exported, so `as Email` can't happen elsewhere in the app
export function email(raw: string): Result<Email, "invalid_email"> {
  const trimmed = raw.trim().toLowerCase();
  return /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(trimmed)
    ? Ok(trimmed as Email)
    : Err("invalid_email");
}

// Everything downstream receives a value that has already been validated:
function sendReceipt(to: Email, p: Payment) { /* no validation needed here */ }
```

**The discipline that makes this work:**
1. The brand and its constructor live in one module.
2. The `as Brand` assertion appears **only** inside the constructor.
3. Enforce it with an ESLint rule if the team is large:
   ```jsonc
   "no-restricted-syntax": ["error", {
     "selector": "TSAsExpression > TSTypeReference[typeName.name=/Id$|^Email$|^Money$/]",
     "message": "Construct branded types via their validating constructor, not `as`."
   }]
   ```

**"Parse, don't validate"** is the name for this idea: a *validator* returns a boolean and leaves you with the same untrusted type; a **parser** returns a *different, narrower type* that carries the proof. Once you hold an `Email`, the check has already happened and cannot be skipped.

---

## 8. Ledger's domain model (the deliverable)

```ts
// ─── Brands ───────────────────────────────────────────────
declare const __brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [__brand]: B };

export type MerchantId = Brand<string, "MerchantId">;
export type PaymentId  = Brand<string, "PaymentId">;
export type RefundId   = Brand<string, "RefundId">;
export type CustomerId = Brand<string, "CustomerId">;
export type IsoDate    = Brand<string, "IsoDate">;
export type IdempotencyKey = Brand<string, "IdempotencyKey">;

// ─── Money ────────────────────────────────────────────────
export type Currency = "usd" | "eur" | "gbp" | "inr" | "jpy";
export type Money<C extends Currency = Currency> = {
  readonly amountMinor: number;
  readonly currency: C;
};

// ─── The payment state machine ────────────────────────────
export type Payment =
  | { status: "requires_payment_method"; id: PaymentId; money: Money; createdAt: IsoDate }
  | { status: "requires_capture"; id: PaymentId; money: Money; createdAt: IsoDate;
      authorizedAt: IsoDate }
  | { status: "succeeded"; id: PaymentId; money: Money; createdAt: IsoDate;
      capturedAt: IsoDate }
  | { status: "failed"; id: PaymentId; money: Money; createdAt: IsoDate;
      failureCode: FailureCode; failedAt: IsoDate }
  | { status: "canceled"; id: PaymentId; money: Money; createdAt: IsoDate;
      canceledAt: IsoDate; canceledBy: "merchant" | "customer" | "system" }
  | { status: "refunded"; id: PaymentId; money: Money; createdAt: IsoDate;
      capturedAt: IsoDate; refunded: Money; refundedAt: IsoDate };

// ─── Operations carry their preconditions ─────────────────
export type Capturable = Extract<Payment, { status: "requires_capture" }>;
export type Refundable = Extract<Payment, { status: "succeeded" }>;
export type Cancelable = Extract<Payment, { status: "requires_payment_method" | "requires_capture" }>;

// ─── Legal transitions, as data the compiler checks ───────
export const TRANSITIONS = {
  requires_payment_method: ["requires_capture", "failed", "canceled"],
  requires_capture:        ["succeeded", "failed", "canceled"],
  succeeded:               ["refunded"],
  failed:                  [],
  canceled:                [],
  refunded:                [],
} as const satisfies Record<Payment["status"], readonly Payment["status"][]>;
// ← add a status to Payment and THIS errors until you declare its transitions

export type NextStatus<S extends Payment["status"]> = typeof TRANSITIONS[S][number];
```

**Read what that last block achieves:** the legal-transition table is `satisfies`-checked against the union, so adding a payment state is a compile error until you say where it can go. The state machine and the type can't drift.

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Count representable vs meaningful states. Close the gap** | The gap is exactly where bugs live |
| **A `return null` for an "impossible" branch means your type is too wide** | Reliable smell |
| **Discriminated union for every multi-state concept** | Each variant carries exactly its own data |
| **`Extract<T, {status: "x"}>` for operations with preconditions** | The precondition becomes uncallable-if-wrong |
| **Brand every ID, unit, and validated string** | Structural typing can't tell two `string`s apart |
| **One validating constructor per brand; `as Brand` appears nowhere else** | The assertion is audited in one place |
| **Money carries its currency in the type** | Cross-currency arithmetic becomes a compile error |
| **`Result` for domain failures; exceptions for bugs** | Don't thread `Result` through twelve layers |
| **Put the rollback target in the optimistic state** | "No way to roll back" becomes unrepresentable |
| **Parse, don't validate — return a narrower type, not a boolean** | The proof travels with the value |
| **`satisfies Record<Union, ...>` for transition tables and lookups** | The table can't drift from the union |

---

## 10. Interview traps

**Q1. "What does 'make illegal states unrepresentable' mean?"**
Design types so the set of representable values equals the set of valid values. **Give the counting argument:** `{isLoading, data?, error?}` permits 8 states of which 3 are meaningful; a discriminated union permits exactly 3. The five extra states are constructible bugs, and every consumer writes defensive code the compiler can't verify.

**Q2. "What's a branded type and what does it cost?"**
An intersection of a primitive with a phantom property (ideally a `unique symbol`) to get nominal behaviour in a structural system. **Runtime cost: zero** — the property never exists. It stops `CustomerId` being passed where `PaymentId` is expected, seconds where milliseconds are expected, and raw strings where sanitised HTML is expected.

**Q3. "How do you stop someone adding USD to EUR?"**
Parameterise `Money` by currency and make the addition function `<C extends Currency>(a: Money<C>, b: Money<C>)`. Mismatched currencies then fail to unify and it's a compile error. That's a financial correctness bug that testing catches unreliably.

**Q4. "`Result` or exceptions?"**
`Result` for expected domain failures the caller must handle — validation, declines, not-found — because the failure becomes part of the signature and the compiler forces a branch. Exceptions for programmer errors and invariant violations. **Don't thread `Result` through every layer**; use it at boundaries.

**Q5. "What's 'parse, don't validate'?"**
A validator returns a boolean and leaves you with the same untrusted type, so the check can be skipped or repeated. A parser returns a *narrower type* that carries the proof — once you hold an `Email`, validation has provably happened. It's the same boundary discipline as the API track's "unknown in, typed out."

**Q6. "Your union model can't express 'stale data while refetching'. What now?"**
Add the state (`{ status: "refetching"; data: T }`) rather than reaching back for optional fields. You had six states, not three — and finding them is the design work. Reverting to optionals re-opens the gap you just closed.

**Q7. "Doesn't this add a lot of boilerplate?"**
Some, and it's front-loaded. The trade: a few extra lines in the type definition vs defensive checks in every consumer, `!` assertions, and a class of bug that only shows up in production. Then the honest limit: *"I'd brand IDs, money and units; I wouldn't brand every string in the app. The test is whether confusing two values would be a real bug."*

**Q8. "How do you stop someone bypassing the brand with `as`?"**
Convention plus tooling: one constructor module, and an ESLint `no-restricted-syntax` rule banning `as SomeBrand` outside it. You can't make it impossible — `as` is always available — so you make it visible and reviewable.

**Q9. "How does this connect to API design?"**
Directly: the API enforces state transitions in SQL (`WHERE status = 'requires_capture'`), the client enforces state shapes in types. Neither trusts the other, and both express the same invariant. And the API's open enums are why the client needs exhaustiveness checks — adding a status server-side must break the client's build, not its rendering.

---

## 11. Build & break

### Build — refactor a wide type
Take this real-world type and close the gap:
```ts
type Upload = {
  file?: File;
  progress?: number;
  url?: string;
  error?: string;
  isUploading: boolean;
  isComplete: boolean;
};
```
1. **Count** the representable states (hint: it's ≥ 64).
2. List the meaningful ones.
3. Rewrite as a discriminated union.
4. Rewrite a consumer and note how many checks disappeared.

### Build — Ledger's domain module
Write `src/domain/` with `brand.ts`, `money.ts`, `ids.ts`, `payment.ts`, `result.ts` from §8. Then write the tests that **must not compile**:
```ts
// @ts-expect-error — cross-currency addition
add(money(100, "usd"), money(100, "eur"));

// @ts-expect-error — raw string as an id
getPayment("pi_123");

// @ts-expect-error — wrong id type
getPayment(customerId("cus_1"));

// @ts-expect-error — capturing a failed payment
capture(failedPayment);

// @ts-expect-error — reading capturedAt off a failed payment
failedPayment.capturedAt;

// @ts-expect-error — seconds where millis are required
setTimeout2(fn, 30 as Seconds);
```
**`@ts-expect-error` is the right tool here:** it fails the build if the line *stops* erroring, so these are real regression tests for your type design. A file full of them is genuinely good evidence in a portfolio repo.

### Break — three experiments
1. **The optional-field trap.** Implement a component against `{isLoading, data?, error?}`. Count your `?.`, `!` and `if` statements. Convert to a union and count again.
2. **The unbranded ID bug.** Write `getPayment(id: string)` and pass a customer ID. It compiles and 404s at runtime. Brand it and watch the same code fail to compile.
3. **The currency bug.** Sum an array of mixed-currency `{amountMinor, currency}` objects with `reduce`. It compiles and produces a meaningless number. Parameterise `Money` by currency and watch it break.

### Explain out loud (90 seconds)
1. The counting argument, with the 8-vs-3 example.
2. What branding is, what it costs, and three things worth branding.
3. When `Result` beats exceptions, and when it doesn't.
4. "Parse, don't validate" in one sentence.
5. How this mirrors the API's state machine.

---

## What's next

You've been *using* types. Next you'll start **computing** them: conditional types and `infer`, which is how `ReturnType`, `Awaited` and every clever library type are built — and how you read the type gymnastics in other people's code.

Next → **[Lesson 10: Conditional types & `infer`](10-conditional-types-and-infer.md)**
