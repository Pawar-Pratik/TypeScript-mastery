# Lesson 05 — Functions, overloads & contextual typing

> **Why this lesson exists:** functions are where TypeScript's inference is most helpful and most surprising. Contextual typing means you often *shouldn't* annotate; overloads are usually the wrong tool for the job people reach for them with; and the rules about parameter counts and `this` produce behaviour that looks like a bug until you know the rule. This lesson makes function typing predictable.

**Time:** ~60 minutes · **Prereq:** Module 1

---

## 1. The idea in one sentence

> **Annotate what goes *in*, let TypeScript infer what comes *out* — and understand contextual typing, because it's why the same callback needs annotations in one position and not in another.**

---

## 2. The basics, and the rules that follow

```ts
function f(a: string, b?: number, c = 3, ...rest: boolean[]): string { return a; }
//           required  optional    default  rest
```

Rules that matter:

**1. Optional parameters must come after required ones.** `(a?: string, b: number)` is an error — there'd be no way to call it.

**2. A parameter with a default is implicitly optional**, and its type is inferred from the default: `c = 3` gives `c: number`. But note the difference:
```ts
function a(x = 3) {}         // x: number,     callable as a()
function b(x?: number) {}    // x: number|undefined, callable as b()
function c(x: number = 3) {} // x: number,     and you can pass undefined explicitly
c(undefined);                 // ✅ → x is 3.  b(undefined) → x is undefined
```
**Passing `undefined` explicitly triggers the default.** That's a JavaScript rule, and it means `x = 3` and `x?: number` behave differently for callers who pass `undefined` — which happens constantly with optional props in React.

**3. Return types: infer by default, annotate at boundaries.**
```ts
function calc(a: number, b: number) { return a * b; }      // ✅ infers number
export function api(): Promise<Payment> { /* ... */ }       // ✅ annotate exported API
```
Why annotate exports: an annotated return type is a **contract** — if the implementation drifts, the error appears in the function, not in the twelve call sites downstream. It also stops an accidental `any` from leaking outward, and it speeds up the compiler (no inference needed across module boundaries).

> **The rule:** infer returns for internal functions (less noise, no drift), annotate returns for anything exported or public. This is the same reasoning as an API contract — the boundary is where you want the check.

---

## 3. Contextual typing — why you sometimes don't annotate

```ts
// ❌ Noisy and redundant
[1, 2, 3].map((n: number): string => n.toString());

// ✅ TypeScript already knows
[1, 2, 3].map(n => n.toString());
//             ^? n: number    — inferred from Array<number>.map's signature
```

**Contextual typing:** when a function expression appears in a position with a known expected type, its parameters are inferred from that type. It's why callbacks rarely need annotations.

It works when the context is known, and fails when it isn't:
```ts
const fn = (n) => n * 2;              // ❌ implicit any — no context
const fn2: (n: number) => number = n => n * 2;    // ✅ context from the annotation
type Mapper = (n: number) => number;
const fn3: Mapper = n => n * 2;                    // ✅ better
```

**The practical pattern:** annotate the *variable*, not the parameters.
```ts
// ❌ annotating each parameter and the return
const handleClick = (e: React.MouseEvent<HTMLButtonElement>): void => {};

// ✅ annotate once, let everything else follow
const handleClick: React.MouseEventHandler<HTMLButtonElement> = e => {};
//                                                              ^? fully typed
```
You'll use this constantly in React ([Lesson 17](../05-ecosystem/17-typescript-with-react.md)).

### Contextual typing and object literals
```ts
type Config = { onDone: (n: number) => void };

const c: Config = { onDone: n => n.toFixed(2) };     // ✅ n: number, inferred
const handler = { onDone: n => n.toFixed(2) };        // ❌ implicit any — no context
```
This is why extracting a handler to a variable *before* passing it can lose type information — the same freshness-style phenomenon as excess property checks in [Lesson 04](../01-foundations/04-objects-and-interfaces.md).

---

## 4. Fewer parameters is fine — the rule that looks like a bug

```ts
type Callback = (value: string, index: number, all: string[]) => void;
const cb: Callback = (value) => console.log(value);     // ✅ perfectly legal
["a"].forEach(v => console.log(v));                      // ✅ ignoring 2 params
```

**A function with fewer parameters is assignable to a type expecting more.** Reason: extra arguments in JavaScript are simply ignored, so it's safe — the callback just doesn't look at them. Without this rule, `arr.forEach(x => ...)` would be an error, which would be absurd.

The reverse is not allowed:
```ts
const bad: Callback = (a, b, c, d) => {};   // ❌ 'd' would always be undefined
```

**The real-world bug this enables** is worth knowing:
```ts
["1", "2", "3"].map(parseInt);      // → [1, NaN, NaN] 😱
// map passes (value, index, array); parseInt takes (string, radix).
// So it calls parseInt("2", 1) — radix 1 is invalid → NaN.
["1","2","3"].map(n => parseInt(n, 10));   // ✅
```
Type-correct, completely wrong. A good illustration that type safety isn't correctness.

---

## 5. Overloads — and why you usually shouldn't

```ts
// Overload signatures (what callers see)
function parse(input: string): object;
function parse(input: string, reviver: (k: string, v: unknown) => unknown): object;
function parse(input: Buffer): object;
// Implementation signature (NOT callable — invisible to callers)
function parse(input: string | Buffer, reviver?: (k: string, v: unknown) => unknown): object {
  const text = typeof input === "string" ? input : input.toString("utf8");
  return JSON.parse(text, reviver);
}
```

Rules:
- The implementation signature must be compatible with all overloads, and is **not itself callable**.
- Overloads are resolved **top to bottom, first match wins** — so order matters: put more specific signatures first.
- The implementation body gets no narrowing from which overload was chosen; you must narrow yourself.

### When overloads are genuinely right
**When the return type depends on the argument in a way a union can't express.**
```ts
// The classic, and it's a real one:
function createElement(tag: "a"): HTMLAnchorElement;
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: string): HTMLElement;
```

### When they're wrong — three better alternatives

**1. A union parameter, when the return type doesn't change:**
```ts
// ❌ pointless overloads
function log(x: string): void;
function log(x: number): void;
function log(x: string | number): void { console.log(x); }

// ✅
function log(x: string | number): void { console.log(x); }
```

**2. Generics, when the return relates to the input:**
```ts
// ❌ doesn't scale, and loses precision
function first(arr: string[]): string | undefined;
function first(arr: number[]): number | undefined;

// ✅ one signature, works for every T
function first<T>(arr: readonly T[]): T | undefined { return arr[0]; }
```

**3. Conditional return types, for genuinely dependent returns** ([Lesson 10](../03-type-level/10-conditional-types-and-infer.md)):
```ts
function get<T extends boolean>(single: T): T extends true ? Payment : Payment[];
```
Powerful, and harder to read — reach for it only when overloads genuinely can't express it.

### The overload trap: unions don't distribute
```ts
function parse(x: string): object;
function parse(x: Buffer): object;
function parse(x: any): object { /* ... */ }

const input: string | Buffer = getInput();
parse(input);    // ❌ No overload matches this call
```
Overload resolution tries each signature against the *whole* argument type; `string | Buffer` matches neither. Adding a `string | Buffer` overload fixes it — which is a strong hint that a plain union parameter was the right design all along.

> **The rule:** *"reach for a union first, generics second, conditional types third, and overloads only when the return type varies by literal argument in a way nothing else expresses. Overloads look like the obvious tool and they're usually the fourth-best one."*

---

## 6. `this` typing

```ts
// A `this` parameter — erased at compile time, purely for checking
function handler(this: HTMLButtonElement, e: MouseEvent) {
  this.disabled = true;
}
button.addEventListener("click", handler);   // ✅ `this` is checked

// ThisType<T> in an object literal — how Vue-style APIs are typed
const obj = {
  count: 0,
  increment() { this.count++; },     // ✅ `this` inferred as the object
};

// Arrow functions capture `this` lexically and CANNOT have a `this` parameter
const bad = (this: Window) => {};    // ❌
```

**The classic bug, and why it's a class bug not a TypeScript bug:**
```ts
class Counter {
  count = 0;
  increment() { this.count++; }
  incrementArrow = () => { this.count++; };   // bound at construction
}
const c = new Counter();
const f = c.increment;
f();                    // 💥 'this' is undefined
const g = c.incrementArrow;
g();                    // ✅ works — `this` was captured
```
TypeScript can catch some of this with `strictBindCallApply` and a `this` parameter, but the general case isn't caught. **This is why React class components needed `this.handleClick = this.handleClick.bind(this)`** — and one of several reasons hooks won.

---

## 7. Function types in practice

```ts
// Two ways to write a function type
type Fn1 = (a: string) => number;
interface Fn2 { (a: string): number }         // call signature — allows extra props too

// A callable object: a function with properties
interface Middleware {
  (req: Request, res: Response, next: NextFunction): void;
  name: string;
  enabled: boolean;
}

// Constructor types
type Ctor<T> = new (...args: any[]) => T;
function instantiate<T>(C: Ctor<T>): T { return new C(); }

// Extracting parts of a function type (Lesson 12)
type P = Parameters<typeof fetch>;            // [input: RequestInfo | URL, init?: RequestInit]
type R = ReturnType<typeof fetch>;            // Promise<Response>
type A = Awaited<ReturnType<typeof fetch>>;   // Response
```

### Typing higher-order functions
```ts
// Preserve the full signature of the wrapped function — this is the idiom
function withLogging<A extends unknown[], R>(fn: (...args: A) => R): (...args: A) => R {
  return (...args) => {
    console.log("calling", fn.name, args);
    return fn(...args);
  };
}

const loggedCharge = withLogging(charge);
//    ^? exactly charge's signature — parameters and return preserved
```
`<A extends unknown[], R>` over the parameter tuple is the pattern for any wrapper, decorator, memoizer or retry helper. Learn it once; you'll use it forever.

```ts
// Async version, with the return unwrapped correctly
function withRetry<A extends unknown[], R>(
  fn: (...args: A) => Promise<R>,
  attempts = 3,
): (...args: A) => Promise<R> {
  return async (...args) => {
    let last: unknown;
    for (let i = 0; i < attempts; i++) {
      try { return await fn(...args); } catch (e) { last = e; }
    }
    throw last;
  };
}
```

---

## 8. Assertion functions and never-returning functions

```ts
// A type predicate: "if this returns true, x is a Payment"
function isPayment(x: unknown): x is Payment { /* ... */ }

// An assertion function: "if this returns at all, x is a Payment"
function assertPayment(x: unknown): asserts x is Payment {
  if (!isPayment(x)) throw new Error("not a payment");
}

// A generic invariant assertion — extremely useful
function assert(cond: unknown, msg?: string): asserts cond {
  if (!cond) throw new Error(msg ?? "Assertion failed");
}

const p: unknown = load();
assertPayment(p);
p.amountMinor;              // ✅ narrowed for the rest of the scope

// A never-returning function narrows by exclusion
function fail(msg: string): never { throw new Error(msg); }
const x: string | undefined = maybe();
if (!x) fail("required");
x.toUpperCase();            // ✅ narrowed to string
```

**Two gotchas with assertion functions:**
1. They **must have an explicit type annotation** at the declaration — you can't infer `asserts`. So `const assert = (c: unknown): asserts c => {}` fails; you need a `function` declaration or an explicitly-typed variable.
2. **TypeScript trusts them completely.** An assertion function whose body doesn't actually check is a silent lie — the same category of risk as `as`.

Full treatment of narrowing in [Lesson 07](07-narrowing-and-control-flow.md).

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Annotate parameters; infer internal return types** | Parameters are the contract; inferred returns can't drift |
| **Annotate return types on exported functions** | Errors surface in the function, not at twelve call sites; and blocks `any` leaking outward |
| **Annotate the *variable*, not each parameter, for callbacks** | `const h: MouseEventHandler = e => {}` beats annotating `e` |
| **Union first, generics second, conditional types third, overloads last** | Overloads look obvious and are usually the fourth-best tool |
| **Order overloads specific → general** | First match wins |
| **`<A extends unknown[], R>` for wrappers** | Preserves the wrapped signature exactly |
| **Accept `readonly T[]` in parameters** | Wider accepted set; documents non-mutation |
| **Never pass a multi-arg function directly as a callback** | `map(parseInt)` — extra arguments get bound |
| **Use `asserts` functions for invariants, but make sure they really check** | The compiler trusts them unconditionally |
| **Prefer `function` declarations for hoisting-sensitive code, arrows for callbacks** | And arrows for class fields when you need bound `this` |

---

## 10. Interview traps

**Q1. "Why is `(a) => a` an implicit `any` here but not there?"**
Contextual typing. In a position with a known expected function type, parameters are inferred from that type. With no context (a bare `const fn = a => a`), there's nothing to infer from. Fix by annotating the variable's type rather than each parameter.

**Q2. "Why can I pass a 1-parameter function where a 3-parameter one is expected?"**
Because extra arguments are ignored in JavaScript, so it's safe — and without the rule, `arr.forEach(x => ...)` wouldn't compile. Then land the famous consequence: `["1","2","3"].map(parseInt)` is `[1, NaN, NaN]`, because `map` passes the index as `parseInt`'s radix. **Type-correct, entirely wrong.**

**Q3. "When would you use an overload instead of a union?"**
Only when the **return type depends on which argument shape was passed** in a way a union can't express — `createElement("input")` returning `HTMLInputElement`. If the return type is the same, a union parameter is strictly better; if the return relates to the input generically, use a generic.

**Q4. "What's the downside of overloads?"**
They don't accept union arguments (each signature is tried against the whole argument type, so `string | Buffer` matches neither `string` nor `Buffer`), they duplicate the signature, the implementation gets no narrowing, and they're harder to maintain. That union failure is usually the sign you wanted a union parameter.

**Q5. "Should you annotate return types?"**
Exported/public functions: yes — it's a contract, it keeps errors local, and it prevents `any` leaking. Internal functions: usually no — inference is accurate and annotations drift. Some teams enforce `explicit-module-boundary-types` in ESLint, which is exactly this rule.

**Q6. "Type predicate vs assertion function?"**
A predicate (`x is T`) returns a boolean you branch on. An assertion (`asserts x is T`) narrows for the rest of the scope by throwing otherwise. Use predicates in conditionals and `.filter()`; use assertions for invariants at the top of a function. Both are **trusted unconditionally**, so a wrong one is as dangerous as `as`.

**Q7. "How do you type a function that wraps another and preserves its signature?"**
```ts
function wrap<A extends unknown[], R>(fn: (...a: A) => R): (...a: A) => R
```
Generic over the parameter tuple and the return type. This is the pattern for logging, retry, memoize, throttle and decorators.

**Q8. "Why did React class components need `.bind(this)`?"**
Because extracting a method loses its receiver — `const f = c.increment; f()` gives `this === undefined` in strict mode. Arrow class fields capture `this` at construction, which is the fix. TypeScript can catch some cases with a `this` parameter but not the general case.

**Q9. "What's the difference between `x = 3` and `x?: number` in a parameter?"**
With a default, passing `undefined` **triggers the default** (`f(undefined)` → `3`) and the parameter's type is `number`. With `?`, the type includes `undefined` and no substitution happens. This matters constantly for optional React props forwarded down.

---

## 11. Build & break

### Build — Ledger's function helpers
`src/lib/fn.ts`:
```ts
/** Preserve the wrapped signature exactly. */
export function withTiming<A extends unknown[], R>(
  name: string, fn: (...args: A) => R,
): (...args: A) => R {
  return (...args) => {
    const t0 = performance.now();
    try { return fn(...args); }
    finally { metrics.observe("fn_duration_ms", performance.now() - t0, { name }); }
  };
}

/** Async retry with full jitter — mirrors API Lesson 18, now type-safe. */
export function withRetry<A extends unknown[], R>(
  fn: (...args: A) => Promise<R>,
  opts: { attempts?: number; baseMs?: number; retryable?: (e: unknown) => boolean } = {},
): (...args: A) => Promise<R> {
  const { attempts = 3, baseMs = 200, retryable = () => true } = opts;
  return async (...args) => {
    let last: unknown;
    for (let i = 1; i <= attempts; i++) {
      try { return await fn(...args); }
      catch (e) {
        last = e;
        if (!retryable(e) || i === attempts) throw e;
        await sleep(Math.random() * baseMs * 2 ** (i - 1));
      }
    }
    throw last;
  };
}

/** Invariant assertion — must be a function declaration with an explicit annotation. */
export function assert(cond: unknown, msg?: string): asserts cond {
  if (!cond) throw new Error(msg ?? "Assertion failed");
}

/** Exhaustiveness helper — call it in a default branch. */
export function assertNever(x: never, msg = "Unexpected variant"): never {
  throw new Error(`${msg}: ${JSON.stringify(x)}`);
}
```
Verify by hovering: `withRetry(chargeCard)` must have **exactly** `chargeCard`'s signature. If it shows `(...args: unknown[]) => Promise<unknown>`, your generics are wrong.

### Break — five experiments
1. **`map(parseInt)`.** Run it. Then run `map(n => parseInt(n, 10))`. Explain the difference from the arity rule.
2. **Lose contextual typing.** Extract an inline callback into a `const` without a type annotation and watch `noImplicitAny` fire. Fix it by annotating the variable's *function type*, not its parameters.
3. **Overload union failure.** Write two overloads (`string`, `Buffer`) and call with a `string | Buffer` variable. Read the error. Then replace the overloads with a single union parameter.
4. **Unbound `this`.** Write a class with a method and an arrow field; extract both and call them. One throws.
5. **A lying assertion.** Write `function assertPayment(x: unknown): asserts x is Payment {}` with an **empty body**. Call it on a string, then access `.amountMinor`. It compiles and crashes — proving the compiler trusts assertions blindly.

### Explain out loud (60 seconds)
1. What contextual typing is and when it fails.
2. Why fewer parameters is allowed, and the `parseInt` consequence.
3. When an overload is right, and the three things to try first.
4. The wrapper generic signature, and why it's shaped that way.
5. Predicate vs assertion function, and the risk both share.

---

## What's next

The most important lesson in the track. You now know that TypeScript allows unsafe code in specific places — next you'll learn **exactly why**, via structural typing and variance. This is the lesson that turns "TypeScript is weird sometimes" into "TypeScript is following a rule I can state."

Next → **[Lesson 06: Structural typing, assignability & variance](06-structural-typing-and-variance.md)**
