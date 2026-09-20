# Lesson 06 — Structural typing, assignability & variance

> **Why this lesson exists:** this is the most important lesson in the track. Every *"why does this compile when it's clearly unsafe?"* and every *"why won't this compile when it's obviously fine?"* question has its answer here. Variance is also the highest-signal TypeScript interview topic — very few candidates can explain why function parameters are contravariant, and the ones who can are visibly operating at a different level.

**Time:** ~80 minutes · **Prereq:** Lessons 03, 05

---

## 1. The idea in one sentence

> **Type compatibility is decided by *shape*, not by name — and when types are nested inside other types (arrays, functions, promises), the direction of compatibility flips depending on the position, which is what "variance" means.**

---

## 2. Structural typing: shape is identity

```ts
interface Point { x: number; y: number }
class Vector { constructor(public x: number, public y: number) {} }

function length(p: Point) { return Math.hypot(p.x, p.y); }

length(new Vector(3, 4));               // ✅ Vector has x and y. Done.
length({ x: 3, y: 4 });                 // ✅
length({ x: 3, y: 4, z: 5 });           // ❌ excess property check (fresh literal)
const v = { x: 3, y: 4, z: 5 };
length(v);                               // ✅ not fresh — extra property is fine
```

**Nothing declares that `Vector` implements `Point`.** It just has the right shape. Compare with Java/C#, where a class must *say* `implements Point`.

| | Structural (TypeScript, Go) | Nominal (Java, C#, Rust) |
|---|---|---|
| Compatibility by | **Shape** | **Declared name** |
| Retrofitting an interface onto existing types | **Free** | Requires modifying the type |
| Accidental compatibility | **Possible** — `UserId` and `PostId` are both `string` | Impossible |
| Duck typing / mocks in tests | **Trivial** | Needs interfaces everywhere |

**Why TypeScript chose structural:** it had to describe *existing JavaScript*, where nothing declares anything. A nominal system couldn't type jQuery, Node's callbacks, or a plain object literal. Structural typing was the only option, and it turns out to be pleasant — you can write a function taking `{ id: string }` and everything with an `id` works.

### The cost of structural typing
```ts
type UserId = string;
type PostId = string;

function getUser(id: UserId) {}
const postId: PostId = "post_123";
getUser(postId);       // ✅ compiles. Both are just `string`. This is a real bug.
```
Type *aliases* are not new types — they're names for existing ones. Getting real nominal behaviour requires **branding**, which is [Lesson 09](../03-type-level/09-illegal-states-unrepresentable.md).

### Private members do create nominal behaviour
The one exception, worth knowing:
```ts
class A { private secret = 1; }
class B { private secret = 1; }
let a: A = new B();     // ❌ Types have separate declarations of a private property 'secret'
```
Classes with `private` or `protected` members compare **nominally** for those members — because two independently-declared private fields can't be assumed interchangeable. It's the only place TypeScript is nominal by default (besides `enum`).

---

## 3. Object assignability: the rules

`Source` is assignable to `Target` if, for every member of `Target`:
1. `Source` has that member, **and**
2. the member's type is assignable, **and**
3. optionality is compatible (a required target member can't be satisfied by an optional source member)

```ts
type Target = { a: string; b?: number };

{ a: "x" }                        // ✅ b is optional
{ a: "x", b: 1, c: true }         // ✅ extra props ok (if not a fresh literal)
{ a: "x", b: "1" }                // ❌ b's type doesn't match
{ b: 1 }                          // ❌ missing required a
{ a: "x" } as { a?: string }      // ❌ optional source, required target
```

### Unions and intersections
```ts
// To a union: assignable if it matches ANY member
let u: string | number = "x";              // ✅

// FROM a union: must be assignable to the target for EVERY member
function f(x: string | number) {
  const s: string = x;                     // ❌ number isn't a string
}

// To an intersection: must satisfy ALL members
type Both = { a: string } & { b: number };
const b: Both = { a: "x" };                // ❌ missing b
```

---

## 4. Variance — the core of the lesson

**Variance answers:** if `Dog` is assignable to `Animal`, is `F<Dog>` assignable to `F<Animal>`?

Four possible answers, and each occurs in TypeScript:

| Variance | Rule | Where |
|---|---|---|
| **Covariant** | `Dog → Animal` ⟹ `F<Dog> → F<Animal>` (same direction) | Return types, readonly properties, `readonly T[]`, `Promise<T>` |
| **Contravariant** | `Dog → Animal` ⟹ `F<Animal> → F<Dog>` (**flipped**) | Function **parameters** |
| **Invariant** | Neither direction | Mutable properties, mutable `T[]` (in a sound system) |
| **Bivariant** | Both directions (**unsound**) | Method parameters, and arrays in practice |

```ts
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }
// Dog is assignable to Animal (it has everything Animal has)
```

### Covariance — the intuitive case

```ts
type Getter<T> = () => T;

const dogGetter: Getter<Dog> = () => new Dog();
const animalGetter: Getter<Animal> = dogGetter;   // ✅ COVARIANT
```

Why it's safe: a caller expecting `Animal` calls it, gets a `Dog`, and a `Dog` *is* an `Animal`. Every use is valid. **Return positions are covariant.**

### Contravariance — the counter-intuitive case, and the one interviewers ask about

```ts
type Handler<T> = (value: T) => void;

const animalHandler: Handler<Animal> = (a) => console.log(a.name);
const dogHandler: Handler<Dog> = (d) => console.log(d.breed);

const h1: Handler<Dog> = animalHandler;      // ✅ CONTRAVARIANT — safe
const h2: Handler<Animal> = dogHandler;      // ❌ unsafe (with strictFunctionTypes)
```

**Work through why, because this is the explanation that matters:**

`h1`: someone will call `h1(someDog)`. Inside, `animalHandler` only touches `.name`, which every `Dog` has. **Safe.** A function that can handle *any* animal can certainly handle a dog.

`h2`: someone will call `h2(someCat)` — legal, because `h2` is typed as `Handler<Animal>`. Inside, `dogHandler` reads `.breed`. **Cats have no breed. Crash.**

> **The one-liner to remember:** *"a function that accepts **more** is substitutable where **less** is expected."* You can always pass a more general handler where a specific one is needed, never the reverse. Parameters flow *in*, so their compatibility flows *backwards* — that's contravariance.

The everyday version:
```ts
// A function that handles any Error can be used where a TypeError handler is expected
type ErrorHandler = (e: Error) => void;
type TypeErrorHandler = (e: TypeError) => void;
const generic: ErrorHandler = e => console.log(e.message);
const specific: TypeErrorHandler = generic;   // ✅ — generic handles more, so it fits
```

### Function assignability, complete

A function `S` is assignable to `T` when:
- **Parameters are contravariant:** each of `T`'s parameter types is assignable to `S`'s (backwards)
- **Return type is covariant:** `S`'s return is assignable to `T`'s (forwards)
- **`S` may take fewer parameters** than `T` (Lesson 05)

```ts
type T = (a: Animal, b: string) => Animal;

const ok:  T = (a: Animal) => new Dog();        // ✅ fewer params, narrower return
const bad: T = (a: Dog) => new Dog();           // ❌ param too narrow
const bad2: T = (a: Animal) => "not an animal"; // ❌ return incompatible
```

---

## 5. `strictFunctionTypes` and the method exception

Here's the deliberate hole, and knowing it is a genuine differentiator.

```ts
interface A { fn: (x: Dog) => void }      // PROPERTY syntax   → contravariant ✅
interface B { fn(x: Dog): void }          // METHOD shorthand  → BIVARIANT ⚠️

declare const a: A;
declare const b: B;

const a2: { fn: (x: Animal) => void } = a;   // ❌ correctly rejected
const b2: { fn(x: Animal): void } = b;       // ✅ ALLOWED — unsound!
```

**`strictFunctionTypes` only applies to function-type *properties*, never to method shorthand.** That's documented and intentional.

**Why the exception exists** — this is the interesting part:

```ts
// If method parameters were contravariant, this would break:
interface Array<T> {
  push(...items: T[]): number;
  concat(...items: ConcatArray<T>[]): T[];
}
const dogs: Dog[] = [];
const animals: Animal[] = dogs;       // relies on arrays being covariant
```

The entire standard library — `Array<T>`, `Promise<T>`, and thousands of DOM interfaces — was written assuming bivariant methods. Enforcing contravariance would break `Array<Dog>` being usable as `Array<Animal>`, which is idiomatic JavaScript that everyone relies on. **TypeScript chose compatibility with existing patterns over soundness.**

### The consequence: arrays are unsoundly covariant

```ts
const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;       // ✅ allowed (covariant)
animals.push(new Cat());              // ✅ allowed — Cat is an Animal
dogs[1].breed;                         // 💥 runtime: undefined — it's a Cat
```

Mutable arrays should be **invariant** (you both read and write `T`), but TypeScript treats them as covariant because it's overwhelmingly convenient and the bug is rare in practice.

**The fix, when you care:** use `readonly`.
```ts
function countNames(animals: readonly Animal[]) {   // ✅ can't push — genuinely safe
  return animals.map(a => a.name).length;
}
countNames(dogs);      // ✅ safe: read-only means covariance IS sound here
```
This is the technical reason behind Lesson 03's advice to accept `readonly T[]` in parameters: **read-only makes covariance sound.** Covariance is only unsafe when writes are possible.

> **The answer to give:** *"Arrays are covariant in TypeScript, which is unsound — you can widen `Dog[]` to `Animal[]` and then push a `Cat`. It's deliberate, because the standard library and idiomatic JS depend on it. `readonly T[]` is the sound version, because covariance is only unsafe in the presence of writes. Related: method-shorthand parameters are bivariant while function-property parameters are contravariant, and `strictFunctionTypes` only checks the latter — for the same compatibility reason."*

---

## 6. Variance annotations (`in` / `out`)

TypeScript 4.7 added explicit variance annotations for generics.

```ts
interface Getter<out T> { get(): T }              // covariant
interface Setter<in T> { set(v: T): void }         // contravariant
interface Box<in out T> { get(): T; set(v: T): void }   // invariant
```

Two things they're for:
1. **Documentation and verification.** TypeScript *checks* your annotation is correct — if you mark something `out` but use `T` in a parameter, it errors. That's a useful design constraint.
2. **Compiler performance.** Variance is normally computed structurally, which is expensive for deeply nested generics. Annotations let the checker short-circuit. This matters in large libraries ([Lesson 18](../05-ecosystem/18-tooling-and-performance.md)).

You rarely need to write these in application code. **Know what they mean when you read a library's types** — and note that the `in`/`out` naming maps exactly to positions: `in` = parameter position, `out` = return position.

---

## 7. Assignability of the special types

```ts
// any — both directions. This is what makes it dangerous.
let a: any = 1; const s: string = a;       // ✅ both ways

// unknown — top type: from everything, to nothing
let u: unknown = 1; const s2: string = u;  // ❌

// never — bottom type: to everything, from nothing
let n: never = 1 as never; const s3: string = n;   // ✅

// void — special in return position (Lesson 03)
const f: () => void = () => 42;            // ✅
```

### `unknown` in a parameter is contravariantly maximal
```ts
type Handler = (x: string) => void;
const universal = (x: unknown) => console.log(x);
const h: Handler = universal;      // ✅ — a handler accepting ANYTHING fits anywhere
```
That's contravariance at its logical extreme, and it's why `(x: unknown) => void` is the most broadly-assignable handler type. Useful for generic logging/telemetry callbacks.

### Optional properties and variance
```ts
type A = { x?: string };
type B = { x: string | undefined };
const a: A = {};
const b: B = a;         // ❌ (property x is missing)  — see Lesson 04
const a2: A = { x: undefined } as B;   // ✅ (or ❌ with exactOptionalPropertyTypes)
```

---

## 8. Getting nominal typing when you need it

Structural typing's cost is accidental compatibility. Three escape hatches:

```ts
// 1. Branding via intersection with a phantom property — the standard approach
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;

function getUser(id: UserId) {}
const pid = "post_1" as PostId;
getUser(pid);              // ❌ — different brands. Exactly what we wanted.

// 2. A unique symbol — stronger, since the key can't be forged
declare const brand: unique symbol;
type Branded<T, B> = T & { readonly [brand]: B };

// 3. A private class member — real nominal typing, with runtime cost
class UserId2 { private _ = 0; constructor(public value: string) {} }
```

**Branding costs nothing at runtime** (`__brand` never exists; it's a compile-time fiction) and it's how you'd stop the `UserId`/`PostId` bug, and how you'd stop adding USD to EUR. Full treatment in [Lesson 09](../03-type-level/09-illegal-states-unrepresentable.md).

---

## 9. Production rules

| Rule | Why |
|---|---|
| **Accept `readonly T[]` in parameters unless you mutate** | Makes covariance sound; accepts more callers |
| **Prefer function-property syntax over method shorthand in your own interfaces** | Property parameters are contravariant (checked); method parameters are bivariant (unchecked) |
| **Brand your ID types and units** | Structural typing can't distinguish two `string`s otherwise |
| **Never widen a mutable array to a supertype** | `const animals: Animal[] = dogs` is a live footgun |
| **Use `(x: unknown) => void` for generic callbacks** | Maximally assignable, and forces the handler to narrow |
| **When a "safe" assignment is rejected, check variance before reaching for `as`** | The compiler is usually right, and the assertion hides a real bug |
| **Read `in`/`out` annotations in library types as position markers** | `in` = parameter, `out` = return |
| **Remember that `private` members make classes nominal** | It's the reason a "compatible" class assignment sometimes fails |

---

## 10. Interview traps

**Q1. "Structural vs nominal typing?"**
Compatibility by shape vs by declared name. TypeScript had to be structural to describe existing JavaScript, where nothing declares anything. The cost is accidental compatibility (`UserId` and `PostId` are both `string`), fixed by branding. Bonus: `private` class members and `enum` are the only places TypeScript is nominal.

**Q2. "Explain covariance and contravariance."**
Covariant = same direction (return types, `readonly T[]`, `Promise<T>`). Contravariant = flipped (function parameters). Then **the reasoning**: a `Handler<Animal>` is assignable to `Handler<Dog>` because it will only ever be called with dogs, and it handles all animals. The reverse would let a `Cat` reach a function reading `.breed`. *"A function accepting more is substitutable where less is expected."*

**Q3. "Why does this compile?"**
```ts
const handlers: Array<(e: Event) => void> = [];
const onClick = (e: MouseEvent) => console.log(e.clientX);
handlers.push(onClick);
```
The signature-in-question is `(e: MouseEvent) => void` being pushed where `(e: Event) => void` is expected — parameters going *narrower*, which contravariance should reject. It compiles because **`Array.push` is a method, and method parameters are bivariant.** The unsoundness is real: `handlers[0](new KeyboardEvent("keydown"))` would read `.clientX` off a keyboard event. And `strictFunctionTypes` doesn't help, by design, because the standard library depends on bivariant methods.

**Q4. "Is `Dog[]` assignable to `Animal[]`? Should it be?"**
Yes, and no. Arrays are covariant in TypeScript, which is unsound because you can then `push(new Cat())` and corrupt the original `Dog[]`. A sound system makes mutable arrays invariant. TypeScript chose convenience — the pattern is idiomatic and the bug is rare. `readonly Animal[]` is the sound version.

**Q5. "What does `strictFunctionTypes` do, and what's its exception?"**
It makes function *parameter* types contravariant instead of bivariant — but only for function-type **properties**, not **method shorthand**. The exception exists so `Array<T>` and thousands of DOM interfaces keep working.

**Q6. "How do you get nominal typing in TypeScript?"**
Branded types — intersect with a phantom property (`string & { __brand: "UserId" }`) or a `unique symbol` key. Zero runtime cost. Or a class with a `private` member, which is genuinely nominal but has runtime weight.

**Q7. "Why can a function with fewer parameters be assigned to one with more?"**
Extra arguments are ignored in JavaScript, so it's safe — and it's required for `arr.forEach(x => ...)` to work. The famous consequence is `map(parseInt)`.

**Q8. "Why is `(x: unknown) => void` assignable to `(x: string) => void`?"**
Contravariance taken to its limit: a handler that accepts *any* value can certainly accept a string. `unknown` is the top type, so it's the maximally-assignable parameter type.

**Q9. "Two classes with identical shapes but a `private` field each — assignable?"**
No. Classes with private/protected members compare nominally for those members. It's the one place TypeScript's default is nominal, and it's why some "shape-identical" assignments unexpectedly fail.

**Q10. "You get a variance error you think is wrong. What do you do?"**
Don't reach for `as`. Work out which position the type appears in: a rejected narrowing in a parameter position is contravariance protecting you from a real bug. Usually the right fixes are widening the parameter, making the container `readonly`, or genuinely making the function accept the wider type.

---

## 11. Build & break

### Build — the variance lab
`src/variance.ts`. **Predict every line before checking.**
```ts
class Animal { name = ""; }
class Dog extends Animal { breed = ""; }
class Cat extends Animal { livesLeft = 9; }

// Return position — covariant
type Get<T> = () => T;
let g1: Get<Animal> = (() => new Dog()) as Get<Dog>;        // ?
let g2: Get<Dog>    = (() => new Animal()) as Get<Animal>;  // ?

// Parameter position — contravariant (property syntax)
type Fn<T> = { call: (x: T) => void };
let f1: Fn<Dog>    = null as any as Fn<Animal>;             // ?
let f2: Fn<Animal> = null as any as Fn<Dog>;                // ?

// Method shorthand — BIVARIANT
type M<T> = { call(x: T): void };
let m1: M<Dog>    = null as any as M<Animal>;               // ?
let m2: M<Animal> = null as any as M<Dog>;                  // ?  ← the unsound one

// Arrays
const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;                              // ?
const ro: readonly Animal[] = dogs;                          // ?

// Promise
let p1: Promise<Animal> = Promise.resolve(new Dog());        // ?
```
For every `✅` that surprises you, write one sentence naming the rule.

### Break — three unsoundness demos, run them
```ts
// 1. Array covariance corrupts the original
const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;
animals.push(new Cat());
console.log(dogs[1].breed);          // 💥 undefined — dogs now contains a Cat
console.log(dogs.map(d => d.breed)); // [.., undefined]

// 2. Method bivariance lets the wrong event through
const handlers: Array<(e: Event) => void> = [];
handlers.push((e: MouseEvent) => console.log(e.clientX));
handlers[0](new KeyboardEvent("keydown"));   // 💥 undefined — no clientX

// 3. readonly makes it safe
function safe(animals: readonly Animal[]) {
  // animals.push(new Cat());          // ❌ correctly rejected
  return animals.map(a => a.name);
}
```
**Then fix #1 and #2** — `readonly Animal[]` for the first, function-property syntax for the second — and confirm the compiler now rejects them.

### Build — branded IDs for Ledger
```ts
declare const __brand: unique symbol;
export type Brand<T, B extends string> = T & { readonly [__brand]: B };

export type MerchantId = Brand<string, "MerchantId">;
export type PaymentId  = Brand<string, "PaymentId">;
export type CustomerId = Brand<string, "CustomerId">;

// Constructors that validate AND brand — the only way to create one
export function paymentId(s: string): PaymentId {
  if (!/^pi_[A-Za-z0-9]{10,}$/.test(s)) throw new Error(`Invalid payment id: ${s}`);
  return s as PaymentId;
}

// Now this bug is impossible:
declare function getPayment(id: PaymentId): Promise<Payment>;
declare const cid: CustomerId;
// getPayment(cid);        // ❌ CustomerId is not PaymentId
// getPayment("pi_123");   // ❌ raw string is not PaymentId — must go through paymentId()
```
Note the design: `as PaymentId` appears **exactly once**, inside a validating constructor. That's the correct use of an assertion — a single audited place where you take responsibility.

### Explain out loud (2 minutes)
1. Structural vs nominal, and why TypeScript had to choose structural.
2. Covariance and contravariance, with the `Handler<Animal>`/`Handler<Dog>` reasoning.
3. Why method parameters are bivariant, and what that costs.
4. Why `Dog[]` → `Animal[]` is unsound, and why `readonly` fixes it.
5. How branding gets you nominal typing for free at runtime.

---

## What's next

You know the assignability rules. Next: **narrowing** — how TypeScript tracks types through your control flow, so you can turn a wide union into a precise type by writing ordinary `if` statements. This is where the type system starts doing genuinely impressive work for you.

Next → **[Lesson 07: Narrowing & control-flow analysis](07-narrowing-and-control-flow.md)**
