# Lesson 16 — Async, errors & the `Result` pattern

> **Why this lesson exists:** async code is where TypeScript's guarantees get thinnest — a function's signature says what it *returns*, never what it *throws*, so the compiler cannot make you handle failure. This lesson covers typing async precisely, the `catch`-is-`unknown` rule (which is correct and surprises everyone), and how to use `Result` where it pays without threading it through your whole codebase.

**Time:** ~65 minutes · **Prereq:** Lessons 09, 15

---

## 1. The idea in one sentence

> **A function's type tells you what it returns, never what it throws — so failure is invisible to the compiler unless you make it part of the return type.**

---

## 2. Typing async precisely

```ts
async function f(): Promise<Payment> { return payment; }        // ✅ annotate exported async fns
async function g() { return payment; }                           // infers Promise<Payment>
async function h(): Promise<void> { await save(); }

// `await` unwraps — and it unwraps recursively
type A = Awaited<Promise<Promise<string>>>;      // string
const x = await Promise.resolve(Promise.resolve(1));   // number, not Promise<number>
```

**The rule people get wrong:** an `async` function *always* returns a `Promise`, and `Promise<Promise<T>>` doesn't exist at runtime — promises auto-flatten. `Awaited<T>` models that flattening, which is why `ReturnType<typeof asyncFn>` gives you `Promise<T>` and you need `Awaited<ReturnType<typeof asyncFn>>` for `T`.

### The `Promise` combinators, typed
```ts
// all — tuple in, tuple out. Rejects on the FIRST failure.
const [p, c] = await Promise.all([getPayment(id), getCustomer(cid)]);
//     ^Payment ^Customer      ← tuple positions preserved

// allSettled — never rejects. Each result is a discriminated union.
const results = await Promise.allSettled([a(), b()]);
for (const r of results) {
  if (r.status === "fulfilled") r.value;    // ✅ narrowed
  else r.reason;                             // `any` — see below
}

// race — first to settle, success OR failure
const winner = await Promise.race([fetchData(), timeout(5000)]);

// any — first to SUCCEED; rejects with AggregateError only if all fail
const fastest = await Promise.any([primary(), replica()]);
```

Two details worth knowing:
- **`Promise.all` rejects on the first failure and abandons the rest** — the others keep running, and their rejections may become unhandled. Use `allSettled` when you want every result.
- **`r.reason` is `any`**, because a rejection can be anything. Narrow it exactly like a `catch` (§3).

```ts
// A typed helper that keeps the tuple AND handles failure — worth having
export async function allSettledTyped<T extends readonly unknown[]>(
  promises: readonly [...{ [K in keyof T]: Promise<T[K]> }],
): Promise<{ [K in keyof T]: Result<T[K], unknown> }> {
  const settled = await Promise.allSettled(promises);
  return settled.map(s =>
    s.status === "fulfilled" ? Ok(s.value) : Err(s.reason)
  ) as { [K in keyof T]: Result<T[K], unknown> };
}
```
That's a mapped type over a tuple ([Lesson 11](../03-type-level/11-mapped-and-template-literal-types.md)) preserving positions — a good example of type-level work earning its place.

---

## 3. `catch` is `unknown` — and that's correct

```ts
try { risky(); }
catch (e) {
  e.message;        // ❌ 'e' is of type 'unknown'   (useUnknownInCatchVariables, in `strict`)
}
```

**Why it's right:** JavaScript can throw *anything*.
```ts
throw "a string";
throw { code: 500 };
throw null;
throw new Error("normal");
// And from a library you don't control, or from a rejected promise with a non-Error value.
```
Pre-`strict`, `e` was `any` and `e.message` was a silent lie — `undefined` for a thrown string, which is how you get log lines reading `Error: undefined`.

### The idiomatic handler
```ts
export function toError(e: unknown): Error {
  if (e instanceof Error) return e;
  if (typeof e === "string") return new Error(e);
  try { return new Error(JSON.stringify(e)); } catch { return new Error(String(e)); }
}

try { await charge(); }
catch (e) {
  if (e instanceof ValidationError) return res.status(422).json(e.toProblem(requestId));
  if (e instanceof ApiError)        return res.status(e.status).json(e.toProblem(requestId));
  const err = toError(e);
  logger.error({ err, requestId }, "unhandled_error");
  return res.status(500).json({ code: "internal_error", request_id: requestId });
}
```
Order matters: **most specific first**, since `ValidationError extends ApiError`.

### `Error.cause` — chain, don't concatenate
```ts
try { await processor.charge(input); }
catch (e) {
  throw new ApiError(502, "upstream_error", "Payment processor failed", undefined, { cause: e });
}
```
`cause` (ES2022) preserves the original error object and its stack. **Node's `console.error` and most loggers print the chain automatically.** The alternative — `new Error("processor failed: " + String(e))` — throws away the stack trace, which is the thing you actually needed at 3am.

```ts
// Walk a cause chain when debugging
export function causeChain(e: unknown): Error[] {
  const out: Error[] = [];
  let cur: unknown = e;
  while (cur instanceof Error && out.length < 10) { out.push(cur); cur = cur.cause; }
  return out;
}
```

---

## 4. The `Result` pattern in practice

From [Lesson 09](../03-type-level/09-illegal-states-unrepresentable.md), now applied to async.

```ts
export type Result<T, E = ApiError> =
  | { ok: true;  value: T }
  | { ok: false; error: E };

export const Ok  = <T,>(value: T): Result<T, never> => ({ ok: true, value });
export const Err = <E,>(error: E): Result<never, E> => ({ ok: false, error });

export type AsyncResult<T, E = ApiError> = Promise<Result<T, E>>;
```

### The critical question: how do you avoid infecting everything?

This is the honest objection to `Result`, and the answer is a **boundary discipline**, not "use it everywhere."

```ts
// ── Layer 1: adapters. Convert throwing code into Result. THIS is where the try/catch lives.
export async function tryAsync<T>(fn: () => Promise<T>): AsyncResult<T, unknown> {
  try { return Ok(await fn()); } catch (e) { return Err(e); }
}

// ── Layer 2: domain functions. Return Result for EXPECTED failures.
export async function chargeCard(input: ChargeInput): AsyncResult<Payment, ChargeError> {
  const validated = ChargeSchema.safeParse(input);
  if (!validated.success) return Err({ code: "invalid_input", issues: validated.error.issues });

  const res = await tryAsync(() => processor.charge(validated.data));
  if (!res.ok) return Err({ code: "processor_unavailable", retryable: true, cause: res.error });

  if (res.value.declined)
    return Err({ code: "card_declined", declineCode: res.value.declineCode, retryable: false });

  return Ok(toPayment(res.value));
}

// ── Layer 3: internal helpers. Plain functions that THROW. No Result plumbing.
function computeFee(money: Money): Money {
  if (money.amountMinor < 0) throw new Error("invariant: negative amount");   // a BUG, not a case
  return money2(Math.round(money.amountMinor * 0.029) + 30, money.currency);
}

// ── Layer 4: the HTTP edge. Result → response. One place.
router.post("/v1/payments", async (req, res) => {
  const r = await chargeCard(req.body);
  if (r.ok) return res.status(201).json(toDto(r.value));

  switch (r.error.code) {
    case "invalid_input":          return res.status(422).json(validationProblem(r.error.issues));
    case "card_declined":          return res.status(402).json(problem(r.error.code, "Card declined"));
    case "processor_unavailable":  return res.status(502).json(problem(r.error.code, "Try again"));
    default:                        return assertNever(r.error);
  }
});
```

**The rule that makes this liveable:** `Result` at **module boundaries** (where a caller must make a decision), exceptions **inside** (where failure means the program is wrong). Threading `Result` through six internal helpers is worse than a `try/catch` at the edge — and saying so is what distinguishes judgement from cargo-culting Rust.

### Why the error type is a union, not `Error`
```ts
type ChargeError =
  | { code: "invalid_input"; issues: ZodIssue[] }
  | { code: "card_declined"; declineCode: DeclineCode; retryable: false }
  | { code: "insufficient_funds"; retryable: false }
  | { code: "processor_unavailable"; retryable: true; cause: unknown }
  | { code: "rate_limited"; retryAfterSec: number; retryable: true };
```
Now `assertNever` forces the HTTP layer to handle every case, and `retryable` — the column from [API Lesson 10](../../API/02-rest-design/10-errors-and-problem-details.md)'s error table — is **in the type**, so a caller cannot forget to branch on it.

---

## 5. Typed async utilities

```ts
/** Timeout that composes with AbortSignal. */
export async function withTimeout<T>(
  fn: (signal: AbortSignal) => Promise<T>,
  ms: number,
  signal?: AbortSignal,
): Promise<T> {
  const combined = signal
    ? AbortSignal.any([signal, AbortSignal.timeout(ms)])    // ES2024 / Node 20+
    : AbortSignal.timeout(ms);
  return fn(combined);
}

/** Retry with full jitter — API Lesson 18, now typed. */
export async function retry<T>(
  fn: (attempt: number) => Promise<T>,
  opts: { attempts?: number; baseMs?: number; maxMs?: number;
          retryable?: (e: unknown) => boolean; signal?: AbortSignal } = {},
): Promise<T> {
  const { attempts = 3, baseMs = 200, maxMs = 10_000, retryable = isTransient, signal } = opts;
  let last: unknown;
  for (let i = 1; i <= attempts; i++) {
    signal?.throwIfAborted();
    try { return await fn(i); }
    catch (e) {
      last = e;
      if (!retryable(e) || i === attempts) throw e;
      await sleep(Math.random() * Math.min(maxMs, baseMs * 2 ** (i - 1)), signal);
    }
  }
  throw last;
}

function isTransient(e: unknown): boolean {
  if (e instanceof ApiError) return e.retryable;
  if (e instanceof DOMException && e.name === "TimeoutError") return true;
  return e instanceof TypeError;              // fetch network failures are TypeError
}

/** Bounded concurrency — a bulkhead (API Lesson 18). */
export async function mapLimit<T, R>(
  items: readonly T[], limit: number, fn: (item: T, i: number) => Promise<R>,
): Promise<R[]> {
  const results = new Array<R>(items.length);
  let next = 0;
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, async () => {
    while (next < items.length) {
      const i = next++;
      results[i] = await fn(items[i]!, i);
    }
  }));
  return results;
}
```

**`AbortSignal` is the piece most people skip and shouldn't.** It's how you cancel a `fetch`, and it's what React's effect cleanup gives you to prevent race conditions ([React Lesson 08](../../React/README.md)). `AbortSignal.timeout()` and `AbortSignal.any()` mean you rarely need to wire one by hand any more.

```ts
// The idiomatic cancellable fetch
async function load(id: PaymentId, signal?: AbortSignal) {
  const res = await fetch(`/v1/payments/${id}`, {
    signal: signal ? AbortSignal.any([signal, AbortSignal.timeout(10_000)])
                   : AbortSignal.timeout(10_000),
  });
  // ...
}
```

---

## 6. The async traps

### Trap 1 — Floating promises
```ts
async function save() { /* ... */ }
save();                      // ⚠️ nobody awaits. A rejection becomes an unhandled rejection.
```
In Node 15+, an unhandled rejection **crashes the process** by default. Catch it with ESLint:
```jsonc
"@typescript-eslint/no-floating-promises": "error"
// Then be explicit when you genuinely mean fire-and-forget:
void save().catch(err => logger.error({ err }, "background save failed"));
```
That `void` + `.catch` is the idiom: it documents the intent *and* handles the rejection.

### Trap 2 — `forEach` with async
```ts
// ❌ Doesn't wait. `forEach` ignores the returned promises entirely.
items.forEach(async item => { await save(item); });
console.log("done");        // logs immediately; nothing is saved yet

// ✅ Sequential
for (const item of items) await save(item);

// ✅ Parallel
await Promise.all(items.map(item => save(item)));

// ✅ Bounded parallel — usually what you actually want
await mapLimit(items, 5, save);
```
**`forEach` with an async callback is silently wrong**, and TypeScript won't stop you because `(item) => Promise<void>` is assignable to `(item) => void` ([Lesson 03](../01-foundations/03-types-as-sets.md)). `no-misused-promises` in ESLint catches it.

### Trap 3 — `async` in a `void` callback
```ts
useEffect(() => fetchData(), []);            // ❌ returns Promise, React expects a cleanup fn
useEffect(() => { void fetchData(); }, []);  // ✅
<button onClick={async () => { await save(); }} />   // ✅ fine — onClick is void-returning
```
Same root cause as Trap 2. React will try to call the returned promise as a cleanup function on unmount.

### Trap 4 — `try/finally` swallowing a return
```ts
async function f(): Promise<number> {
  try { return 1; }
  finally { return 2; }     // ⚠️ returns 2, discarding the try's value AND any exception
}
```
A `return` in `finally` overrides everything. `allowUnreachableCode: false` and ESLint's `no-unsafe-finally` catch it.

### Trap 5 — Sequential awaits that should be parallel
```ts
// ❌ 300ms — three independent calls, one after another
const payment = await getPayment(id);
const customer = await getCustomer(cid);
const refunds = await getRefunds(id);

// ✅ 100ms
const [payment, customer, refunds] = await Promise.all([
  getPayment(id), getCustomer(cid), getRefunds(id),
]);
```
Look for consecutive `await`s where later ones don't use earlier results. This is one of the most common real performance bugs in application code, and it's invisible in a code review unless you're looking for it.

### Trap 6 — Losing the stack across `await`
```ts
// Async stack traces are good in modern V8, but an error created in one tick
// and thrown in another can still lose context. Always chain with `cause`:
catch (e) { throw new ApiError(502, "upstream_error", "…", undefined, { cause: e }); }
```

---

## 7. Async iterators and generators

```ts
// Typed async generator — for streaming/pagination
async function* paginate<T>(
  fetchPage: (cursor?: string) => Promise<{ data: T[]; next_cursor: string | null }>,
): AsyncGenerator<T, void, undefined> {
  let cursor: string | undefined;
  do {
    const page = await fetchPage(cursor);
    yield* page.data;
    cursor = page.next_cursor ?? undefined;
  } while (cursor);
}

// Consume with bounded memory — this is how you export 4M payments
for await (const payment of paginate(c => api.listPayments({ cursor: c }))) {
  await writeRow(payment);
}
```
`AsyncGenerator<Yield, Return, Next>` — the three parameters trip people up; you almost always only care about the first. This pattern is the client-side counterpart of [API Lesson 08](../../API/02-rest-design/08-collections-and-pagination.md)'s cursor pagination: the caller writes an ordinary `for await` loop and never touches a cursor.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Annotate return types on exported async functions** | Prevents `any` leaking through a Promise |
| **`catch (e: unknown)` and narrow — never assume `Error`** | JS can throw anything |
| **Chain with `Error.cause`, never string concatenation** | Preserves the original stack |
| **`Result` at module boundaries; exceptions inside** | Don't thread it through six helpers |
| **Make the error type a discriminated union with a `retryable` field** | The compiler forces callers to branch |
| **`no-floating-promises` and `no-misused-promises` in ESLint** | Unhandled rejections crash Node; `forEach` async is silently wrong |
| **`void fn().catch(log)` for deliberate fire-and-forget** | Documents intent and handles rejection |
| **Never `forEach` with an async callback** | Use `for...of`, `Promise.all`, or `mapLimit` |
| **`Promise.all` for independent awaits** | Consecutive awaits that don't depend on each other are a latency bug |
| **`allSettled` when you need every result** | `all` abandons the rest on first rejection |
| **Pass an `AbortSignal` through every async API you write** | Cancellation and timeouts compose |
| **`AsyncGenerator` for paginated iteration** | Bounded memory, and the caller never sees a cursor |

---

## 9. Interview traps

**Q1. "Why is `catch (e)` typed `unknown`?"**
Because JavaScript can throw anything — a string, `null`, a plain object, a rejected promise's value. `useUnknownInCatchVariables` (part of `strict`) forces you to narrow. Previously it was `any`, so `e.message` was a silent `undefined` — that's how you get `Error: undefined` in logs.

**Q2. "How do you type an error hierarchy?"**
Classes extending `Error` for `instanceof` narrowing (with `setPrototypeOf` if downlevelling), plus `Error.cause` for chaining. For *expected* domain failures, a discriminated union of error objects in a `Result` is better — because then the compiler can force exhaustive handling, which `instanceof` chains can't.

**Q3. "`Result` or exceptions?"**
`Result` for expected failures the caller must handle — the failure becomes part of the signature. Exceptions for programmer errors and invariant violations. **The boundary rule:** `Result` at module edges, exceptions inside. Threading `Result` through every helper is worse than a `try/catch` at the edge.

**Q4. "What's wrong with `items.forEach(async x => await save(x))`?"**
`forEach` ignores returned promises, so nothing is awaited and errors vanish. TypeScript allows it because `() => Promise<void>` is assignable to `() => void`. Use `for...of` (sequential), `Promise.all` (parallel), or a bounded `mapLimit` (usually correct).

**Q5. "What's a floating promise and why does it matter?"**
A promise nobody awaits or catches. In Node 15+ an unhandled rejection **terminates the process**. Enforce `no-floating-promises`; use `void p.catch(log)` when fire-and-forget is genuinely intended.

**Q6. "`Promise.all` vs `allSettled` vs `race` vs `any`?"**
`all`: all succeed or reject on the first failure (and the others keep running). `allSettled`: never rejects, gives a discriminated result per item. `race`: first to settle either way. `any`: first to succeed, `AggregateError` only if all fail. Use `allSettled` when partial success is meaningful.

**Q7. "Three sequential awaits that don't depend on each other — what's the problem?"**
Three round trips instead of one. `Promise.all` them. It's one of the most common real performance bugs in application code and it's invisible unless you look for it.

**Q8. "How do you cancel an in-flight request?"**
`AbortSignal` — `AbortSignal.timeout(ms)` for timeouts, `AbortSignal.any([...])` to combine, and `signal.throwIfAborted()` in loops. Pass a signal through every async API you write. It's also what React's effect cleanup uses to prevent stale-response race conditions.

**Q9. "How would you iterate 4 million paginated records without loading them all?"**
An `async function*` that fetches a page, `yield*`s the items, and follows the cursor — consumed with `for await`. Bounded memory, and the caller never handles a cursor. It's the client half of cursor pagination.

**Q10. "Why does `Awaited<T>` exist?"**
Because promises auto-flatten: `Promise<Promise<T>>` doesn't exist at runtime. `Awaited` models that recursive unwrapping, so `Awaited<ReturnType<typeof asyncFn>>` gives you the actual value type.

---

## 10. Build & break

### Build — Ledger's async toolkit
`src/lib/async.ts`: `tryAsync`, `withTimeout`, `retry` (full jitter + `AbortSignal`), `mapLimit`, `sleep(ms, signal)`, `allSettledTyped`, `paginate`, `toError`, `causeChain`.

Then the typed charge flow from §4 — `ChargeError` as a discriminated union with `retryable`, and an HTTP layer with `assertNever` in the `default`. **Add a new error variant and watch the route handler fail to compile.** That's the whole point.

### Build — the ESLint config that catches the traps
```jsonc
{
  "extends": ["plugin:@typescript-eslint/strict-type-checked"],
  "rules": {
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/no-misused-promises": ["error", {
      "checksVoidReturn": { "arguments": true, "attributes": false }
    }],
    "@typescript-eslint/await-thenable": "error",
    "@typescript-eslint/require-await": "warn",
    "@typescript-eslint/return-await": ["error", "in-try-catch"],
    "no-unsafe-finally": "error"
  }
}
```
`checksVoidReturn.attributes: false` is the pragmatic exception — it lets `onClick={async () => …}` through, which is fine in React.

### Break — six experiments
1. **`forEach` async.** Save 5 items with `forEach` and log "done" after. Watch "done" print first and the saves complete later. Fix three ways and compare.
2. **Floating promise.** `async function f() { throw new Error("boom") } f();` in Node. Watch the process die. Then `void f().catch(log)`.
3. **Thrown string.** `throw "oops"` and handle it with `e.message` (pre-strict style) → `undefined`. Then with the `toError` narrowing.
4. **Lost cause.** Catch an error and re-throw with string concatenation, then with `{ cause: e }`. Compare the printed stacks.
5. **Sequential vs parallel.** Three 100ms calls, awaited sequentially then with `Promise.all`. Time both.
6. **`finally` return.** Reproduce Trap 4 and watch an exception get swallowed.

### Explain out loud (90 seconds)
1. Why `catch` is `unknown` and how you narrow it.
2. The `Result`-at-boundaries rule, and why not everywhere.
3. Three async traps and their fixes.
4. What `AbortSignal` composes.
5. How you'd stream 4M records.

---

## Module 4 complete — checkpoint

- [ ] When a class earns its place; the modules-are-already-DI argument
- [ ] `private` vs `#private`, and the security consequence
- [ ] Three class traps (Error subclassing, lost `this`, field init order)
- [ ] Why plain objects belong at boundaries
- [ ] The module resolution order, and `--traceResolution`
- [ ] What makes a file a module vs a script
- [ ] How to augment `Express.Request`
- [ ] The cost of barrel files; value vs type cycles
- [ ] Where the boundary is — six places
- [ ] Why `as` isn't validation; "parse, don't validate"
- [ ] Why `z.infer` beats a hand-written type + guard
- [ ] Strict-in / tolerant-out
- [ ] Why `catch` is `unknown`
- [ ] `Result` at boundaries, exceptions inside
- [ ] `forEach`-async, floating promises, sequential awaits

---

## What's next

Module 5 connects TypeScript to the rest of your stack. First: **TypeScript with React** — typing components, hooks, events, generics and context without `any` and without fighting the compiler. It's also the bridge into the React track.

Next → **[Lesson 17: TypeScript with React](../05-ecosystem/17-typescript-with-react.md)**
