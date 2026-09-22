# Lesson 13 — Classes, `this`, and OOP in TypeScript

> **Why this lesson exists:** you're coming from Spring, where everything is a class. TypeScript supports classes fully, but idiomatic TypeScript uses them far less than idiomatic Java — and knowing *why*, rather than just following the fashion, is what lets you make the call correctly. This lesson covers the mechanics (including `private` vs `#private`, which is a real runtime difference) and the judgement.

**Time:** ~60 minutes · **Prereq:** Module 3

---

## 1. The idea in one sentence

> **Use a class when you have state and behaviour that genuinely travel together and have a lifecycle; use a plain function and a type for everything else — which, in TypeScript, is most things.**

---

## 2. The mechanics

```ts
class PaymentService {
  // Parameter properties — declare and assign in one step
  constructor(
    private readonly db: Database,
    private readonly processor: Processor,
    public readonly merchantId: MerchantId,
  ) {}

  // Field with an initializer
  private cache = new Map<PaymentId, Payment>();

  // Definite assignment — "trust me, a framework sets this"
  private logger!: Logger;

  // #private — TRULY private at runtime (ES2022)
  #secret = process.env.WEBHOOK_SECRET!;

  // Getter
  get cacheSize(): number { return this.cache.size; }

  // Static
  static create(cfg: Config): PaymentService { /* ... */ }

  // Arrow field — `this` is bound at construction
  handleWebhook = (event: WebhookEvent): void => { this.process(event); };

  // Regular method — `this` depends on the call site
  private process(event: WebhookEvent): void { /* ... */ }
}
```

### `private` vs `#private` — a genuine difference

| | `private x` | `#x` |
|---|---|---|
| Enforced at | **Compile time only** — erased | **Runtime** — a real JS private field |
| Accessible via `obj["x"]` | **Yes** | No |
| Visible in `JSON.stringify`, `Object.keys`, a debugger | **Yes** | No |
| Makes the class nominally typed | **Yes** ([Lesson 06](../02-type-system/06-structural-typing-and-variance.md)) | Yes |
| Subclass can redeclare the same name | No (conflict) | Yes (separate fields) |

```ts
class A { private x = 1; }
console.log(JSON.stringify(new A()));      // {"x":1}   ← private is a lie at runtime
console.log((new A() as any).x);            // 1

class B { #x = 1; }
console.log(JSON.stringify(new B()));       // {}        ← genuinely hidden
console.log((new B() as any).x);            // undefined
```

**The rule:** use `#` for anything that must *actually* stay hidden — secrets, tokens, internal handles you don't want serialized. Use `private` for ordinary encapsulation where compile-time is enough. **And never rely on `private` to keep a secret out of a JSON response** — that's the API track's excessive-data-exposure bug ([API L16](../../API/03-security/16-owasp-and-hardening.md)) wearing a TypeScript costume.

### `abstract`, `implements`, `override`

```ts
abstract class BaseRepo<T, Id> {
  constructor(protected readonly db: Database) {}
  abstract table: string;                              // must be implemented
  abstract toDomain(row: unknown): T;
  async findById(id: Id, merchantId: MerchantId): Promise<T | null> {
    const row = await this.db.one(
      `SELECT * FROM ${this.table} WHERE id=$1 AND merchant_id=$2`, [id, merchantId]);
    return row ? this.toDomain(row) : null;
  }
}

class PaymentRepo extends BaseRepo<Payment, PaymentId> {
  table = "payments";
  override toDomain(row: unknown): Payment { /* ... */ }   // `override` with noImplicitOverride
}
```

**`noImplicitOverride`** ([Lesson 02](../01-foundations/02-tsconfig-deeply.md)) requires the `override` keyword, which catches the bug where a base class renames a method and your "override" silently becomes a new, never-called method. Genuinely worth having on.

**`implements` is a check, not a contract:**
```ts
class Svc implements Runnable { run() {} }
```
It verifies the class satisfies the interface at the declaration — but because TypeScript is structural, *anything* with a `run()` method is already a `Runnable`. `implements` buys you an early, clearly-located error; it doesn't create a type relationship.

⚠️ **`implements` does not infer parameter types**, which surprises people:
```ts
interface Handler { handle(e: Event): void }
class H implements Handler {
  handle(e) {}          // ❌ implicit any — `implements` gives NO contextual typing
}
```

---

## 3. When a class earns its place

### ✅ Genuinely good uses

**1. Stateful objects with a lifecycle**
```ts
class CircuitBreaker {          // state machine with transitions and timers — a class is right
  #state: "closed" | "open" | "half_open" = "closed";
  #failures = 0;
  #openedAt = 0;
  async call<T>(fn: () => Promise<T>): Promise<T> { /* ... */ }
}
```

**2. Error subclasses** — the one place classes are unambiguously correct in TypeScript, because `instanceof` needs a runtime construct:
```ts
export class ApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,
    message: string,
    readonly requestId?: string,
    options?: { cause?: unknown },
  ) {
    super(message, options);
    this.name = "ApiError";
    Object.setPrototypeOf(this, new.target.prototype);   // ← see §4
  }
  get retryable(): boolean { return this.status >= 500 || this.status === 429; }
}

export class ValidationError extends ApiError {
  constructor(readonly fields: readonly FieldError[]) {
    super(422, "validation_failed", "Request validation failed");
    this.name = "ValidationError";
  }
}
```

**3. Resource handles** — connections, pools, subscriptions, anything with `close()`.

**4. Framework requirements** — NestJS, TypeORM, Angular. If the framework wants classes, use classes.

### ❌ Where a class is the wrong reflex

```ts
// ❌ A "service" that's just a namespace for functions
class MathUtils {
  static add(a: number, b: number) { return a + b; }
}
// ✅ Just export functions. Tree-shakeable, testable, no instance to manage.
export const add = (a: number, b: number) => a + b;

// ❌ A data class
class Payment {
  constructor(public id: string, public amount: number) {}
}
// ✅ A type + a plain object. No prototype, serializes cleanly, structurally typed.
type Payment = { id: PaymentId; money: Money };

// ❌ A class holding no state
class PaymentValidator {
  validate(p: unknown): Result<Payment, ValidationError> { /* ... */ }
}
// ✅ A function
export function validatePayment(p: unknown): Result<Payment, ValidationError> { /* ... */ }
```

**Why "just export functions" is genuinely better here**, and this is the argument to be able to make:
- **Tree-shaking works.** An unused exported function is removed from the bundle; an unused static method on an imported class usually isn't.
- **No instance lifecycle** to manage, inject or mock.
- **Testing is direct** — import the function, call it. No construction, no DI container.
- **Plain objects serialize cleanly.** A class instance sent through `JSON.stringify`/`parse` comes back as a plain object with no methods and no prototype — a real bug source at every boundary.

> **The Spring bridge, and it's worth saying out loud:** *"in Spring, classes exist largely because the DI container needs something to instantiate and proxy. In TypeScript, modules are already singletons and imports are already dependency injection — so a stateless service class is a container without contents. I use classes for stateful objects, errors, and resource handles; functions and types for everything else."*

---

## 4. The class traps

### Trap 1 — Extending `Error`
```ts
class MyError extends Error {}
const e = new MyError("boom");
e instanceof MyError;   // false, if targeting ES5!
```
When downlevelling to ES5, `Error` subclassing breaks the prototype chain. Fix: `Object.setPrototypeOf(this, new.target.prototype)` in the constructor, or target ES2015+. **Target ES2022 and it's a non-issue** — but the `setPrototypeOf` line is harmless insurance and you'll see it in library code.

Also: `Error.cause` (ES2022) is the right way to chain errors:
```ts
throw new ApiError(502, "upstream_error", "Processor failed", requestId, { cause: err });
```

### Trap 2 — Losing `this`
```ts
class Service {
  #name = "svc";
  log() { console.log(this.#name); }
}
const s = new Service();
const fn = s.log;
fn();                        // 💥 Cannot read private member from an object whose class did not declare it
setTimeout(s.log, 100);      // 💥 same
setTimeout(() => s.log(), 100);   // ✅
```
Fixes: arrow-function class fields (bound at construction, but one closure *per instance*), or wrap at the call site. **Arrow fields cost memory per instance and aren't on the prototype**, so they can't be overridden by a subclass — a real trade-off, not a free win.

### Trap 3 — Field initialization order
```ts
class A {
  value = this.compute();          // runs BEFORE B's fields exist
  compute() { return 1; }
}
class B extends A {
  multiplier = 2;
  override compute() { return this.multiplier * 10; }   // 💥 multiplier is undefined → NaN
}
new B().value;    // NaN
```
Base-class fields initialize before derived-class fields. **Never call an overridable method from a field initializer or constructor.** `useDefineForClassFields` (default true with ES2022) makes this stricter and more standards-compliant — and more likely to bite you if you were relying on the old behaviour.

### Trap 4 — Class instances don't survive serialization
```ts
const p = new Payment("pi_1", 4999);
const round = JSON.parse(JSON.stringify(p));
round instanceof Payment;    // false
round.format();               // 💥 not a function
```
Anything crossing a network, `localStorage`, a worker, or React Server Component boundary loses its prototype. **This alone is a strong argument for plain objects + functions at every boundary** — and it's why the React track's data layer uses types, not classes.

### Trap 5 — Decorators, and which flavour
```ts
// Legacy (experimentalDecorators) — NestJS, TypeORM, older Angular
@Injectable()
class Service { @Inject() private repo!: Repo; }

// Standard decorators (TS 5.0+, no flag) — different semantics, NOT compatible
function logged<T extends (...a: any[]) => any>(fn: T, ctx: ClassMethodDecoratorContext): T { /* ... */ }
```
**The two are incompatible.** If a framework needs `experimentalDecorators`, you can't also use standard decorators in that project. Know which one you're in.

---

## 5. Idiomatic TypeScript patterns instead of classes

```ts
// ── Instead of a service class: a module of functions ──
// paymentService.ts
export async function create(input: CreatePayment, ctx: Ctx): Promise<Result<Payment, ApiError>> {}
export async function capture(id: PaymentId, ctx: Ctx): Promise<Result<Payment, ApiError>> {}
// import * as payments from "./paymentService"  → namespaced, tree-shakeable

// ── Instead of a builder class: an options object ──
type QueryOptions = { status?: PaymentStatus[]; limit?: number; cursor?: string };
export function listPayments(opts: QueryOptions = {}) {}

// ── Instead of an abstract base: a higher-order function ──
export function makeRepo<T, Id extends string>(cfg: {
  table: string;
  toDomain: (row: unknown) => T;
}) {
  return {
    findById: (id: Id, merchantId: MerchantId) => /* ... */,
    list: (merchantId: MerchantId, opts: QueryOptions) => /* ... */,
  };
}
export const paymentRepo = makeRepo<Payment, PaymentId>({ table: "payments", toDomain });

// ── Instead of a singleton class: a module-level value ──
export const logger = pino({ /* ... */ });      // modules are already singletons

// ── Instead of inheritance: composition ──
type Timestamps = { createdAt: IsoDate; updatedAt: IsoDate };
type Payment = { id: PaymentId; money: Money } & Timestamps;
```

**The closure-based factory (`makeRepo`) is worth noticing:** it gives you private state (the closure), an interface (the returned object), and composition — everything a class gives you — without a prototype, without `this`, and with a result that serializes cleanly. It's the idiomatic TypeScript answer to most "I need a class" impulses.

---

## 6. Ledger's class usage (the worked judgement)

| Thing | Class? | Why |
|---|---|---|
| `ApiError` and subclasses | **Yes** | `instanceof` needs a runtime construct |
| `CircuitBreaker` | **Yes** | Real state machine with a lifecycle |
| `Emitter<EventMap>` | **Yes** | Stateful, with subscription lifecycle |
| `DatabasePool`, `RedisClient` | **Yes** (from libraries) | Resource handles |
| `PaymentService` | **No** | Stateless — a module of functions taking a `ctx` |
| `PaymentRepo` | **No** | A closure factory bound to a tenant ([API L15](../../API/03-security/15-authorization-and-multitenancy.md)) |
| `Payment`, `Money`, `Customer` | **No** | Plain types; they cross the wire |
| `Validator` | **No** | A Zod schema and a function ([Lesson 15](15-the-boundary-and-parsing.md)) |

```ts
// The one error hierarchy Ledger needs
export class ApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,
    message: string,
    readonly details?: Record<string, unknown>,
    options?: { cause?: unknown },
  ) {
    super(message, options);
    this.name = new.target.name;
    Object.setPrototypeOf(this, new.target.prototype);
  }
  get retryable(): boolean {
    return this.status >= 500 || this.status === 429 || this.status === 408;
  }
  toProblem(requestId: string) {
    return {
      type: `https://docs.ledger.dev/errors/${this.code.replaceAll("_", "-")}`,
      title: this.name, status: this.status, detail: this.message,
      code: this.code, request_id: requestId, ...this.details,
    };
  }
}

export class ValidationError extends ApiError {
  constructor(readonly fields: readonly FieldError[]) {
    super(422, "validation_failed", "Request validation failed", { errors: fields });
  }
}
export class ConflictError extends ApiError {
  constructor(code: string, message: string, details?: Record<string, unknown>) {
    super(409, code, message, details);
  }
}
```
Note `toProblem` — the class knows how to serialize itself into the RFC 9457 shape from [API Lesson 10](../../API/02-rest-design/10-errors-and-problem-details.md). That's behaviour genuinely travelling with state, which is the test for whether a class is warranted.

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Classes for stateful objects, errors, and resource handles. Functions for everything else** | Modules are already singletons; imports are already DI |
| **`#private` for anything that must actually be hidden** | `private` is erased and appears in `JSON.stringify` |
| **Never rely on `private` to keep data out of a response** | That's excessive data exposure |
| **`Object.setPrototypeOf(this, new.target.prototype)` in custom `Error`s** | Fixes `instanceof` when downlevelling |
| **Use `Error.cause` to chain, not string concatenation** | Preserves the original stack |
| **`noImplicitOverride` on; use the `override` keyword** | Catches silently-orphaned overrides |
| **Never call an overridable method from a field initializer** | Base fields initialize before derived ones |
| **Arrow fields only when you need bound `this`** | One closure per instance; not overridable |
| **Plain objects + types at every serialization boundary** | Class instances lose their prototype through JSON |
| **`implements` for an early error, not as a type relationship** | It's structural anyway, and it doesn't infer parameter types |
| **Pick one decorator flavour per project** | Legacy and standard decorators are incompatible |

---

## 8. Interview traps

**Q1. "When do you use a class in TypeScript?"**
State + behaviour with a lifecycle; error types (`instanceof` needs a runtime construct); resource handles; framework requirements. **Then the reasoning:** modules are already singletons and imports are already DI, so a stateless service class is a container without contents.

**Q2. "`private` vs `#private`?"**
`private` is compile-time only — erased, visible via `obj["x"]`, and it shows up in `JSON.stringify`. `#` is a real runtime private field. Use `#` for secrets. And note `private` makes the class nominally typed, which is the one place TypeScript isn't structural.

**Q3. "Why does `instanceof` fail on my custom Error?"**
Downlevelling to ES5 breaks the prototype chain for built-in subclassing. Fix with `Object.setPrototypeOf(this, new.target.prototype)` or target ES2015+.

**Q4. "Why did `this` become undefined?"**
The method was extracted from its receiver — `const f = obj.method; f()`. Fix with an arrow class field (bound at construction, costs a closure per instance and isn't overridable) or by wrapping at the call site. This is why React class components needed `.bind(this)`.

**Q5. "What happens to a class instance through `JSON.parse(JSON.stringify(x))`?"**
It becomes a plain object — prototype gone, methods gone, `instanceof` false. **A strong argument for plain objects and functions at every boundary**: network, storage, workers, and server/client component boundaries.

**Q6. "Does `implements` do anything, given structural typing?"**
It's a declaration-site check: you get a clear error at the class rather than at a use site. It creates no type relationship — anything with the right shape already satisfies the interface. And it does **not** provide contextual typing, so method parameters still need annotations.

**Q7. "Why is calling an overridable method from a constructor dangerous?"**
Base-class fields and the base constructor run **before** derived-class field initializers, so an override will see `undefined` for derived fields. The classic `NaN` bug.

**Q8. "Inheritance or composition?"**
Composition, almost always, and TypeScript makes it cheap: intersect types for shared shape, and use higher-order functions or closure factories for shared behaviour. Deep inheritance hierarchies are hard to serialize, hard to tree-shake, and hard to reason about. Reserve `extends` for error hierarchies and genuine is-a relationships.

**Q9. "How would you structure a service layer without classes?"**
A module of exported functions taking an explicit context (`ctx` with a scoped repo, logger, principal), plus closure factories for anything needing bound state. You keep testability (import and call), tree-shaking, and explicit dependencies — and you lose nothing except the DI container you didn't need.

---

## 9. Build & break

### Build — Ledger's error hierarchy
Implement §6's `ApiError`, `ValidationError`, `ConflictError`, `NotFoundError`, `RateLimitError`. Then the central handler:
```ts
export function toProblemResponse(err: unknown, requestId: string) {
  if (err instanceof ApiError) return { status: err.status, body: err.toProblem(requestId) };
  logger.error({ err, requestId }, "unhandled_error");
  return { status: 500, body: { code: "internal_error", request_id: requestId,
                                 detail: "An unexpected error occurred" } };
}
```
Note it mirrors [API Lesson 10](../../API/02-rest-design/10-errors-and-problem-details.md) exactly — same design, now type-safe.

### Build — the same service two ways
Implement a `PaymentService` as (a) a class with injected dependencies and (b) a module of functions taking a `ctx`. Then compare:
- Lines of code
- Test setup required
- What ends up in the bundle if only one function is imported
- What happens when you need two configurations simultaneously

Write your conclusion down. **This is the exercise that converts "TypeScript prefers functions" from a slogan into a judgement you can defend.**

### Break — five experiments
1. **`private` is a lie.** `class A { private secret = "sk_live_x" }` → `JSON.stringify(new A())`. Find the secret. Switch to `#` and repeat.
2. **Broken `instanceof`.** Set `"target": "ES5"`, subclass `Error`, and check `instanceof`. Then add `setPrototypeOf` and re-check.
3. **Lost `this`.** `const fn = instance.method; fn()`. Then with an arrow field. Then check whether a subclass can override the arrow field (it can't).
4. **Field initialization order.** Reproduce the `NaN` bug from Trap 3.
5. **Prototype loss.** Round-trip a class instance through `JSON` and call a method.

### Explain out loud (60 seconds)
1. When a class earns its place in TypeScript, and the module/DI argument.
2. `private` vs `#private`, and the security consequence.
3. Three class traps and their fixes.
4. Why plain objects belong at boundaries.

---

## What's next

You know how to structure code. Next: how TypeScript **finds** it — the module system, `.d.ts` files, ambient declarations, and the fixes for every "cannot find module" and "has no exported member" error you'll ever hit.

Next → **[Lesson 14: Modules, declaration files & ambient types](14-modules-and-declarations.md)**
