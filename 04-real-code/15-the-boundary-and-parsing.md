# Lesson 15 — The boundary: `unknown`, parsing, and Zod

> **Why this lesson exists:** this is the most important *practical* lesson in the track. Everything you've built — branded types, discriminated unions, exhaustive switches — rests on the assumption that your values actually have the shape you claim. At the boundary, nothing enforces that. **Every production TypeScript bug that makes people say "but it type-checked!" lives here.**

**Time:** ~70 minutes · **Prereq:** Lessons 09, 14

---

## 1. The idea in one sentence

> **Types are compile-time claims; the boundary is where claims must become *checks* — so every external input enters as `unknown`, gets parsed once, and emerges as a type you derived from the parser itself.**

---

## 2. The boundary, enumerated

Every one of these is untrusted, and every one is routinely trusted:

```
HTTP responses (fetch, axios)      JSON.parse
localStorage / sessionStorage      URL params & query strings
Form input                          WebSocket messages
process.env                         CLI arguments
File contents (CSV, JSON, YAML)     Database rows (yes — see below)
Third-party SDK callbacks           postMessage / worker messages
Webhook payloads                    `catch` variables
```

**Why database rows count:** your ORM's types describe the schema *you think* exists. After a migration, a manual `UPDATE`, or a column type change, the row may not match. It's a softer boundary than the network — you control it — but it's a boundary.

### The lie everyone writes
```ts
const user: User = await fetch("/api/me").then(r => r.json());
```
`r.json()` returns `Promise<any>`, and `any` assigns to anything. **No check happened.** If the API returns `{ id: 42 }`, then `user.id` is a number, `user.id.toUpperCase()` throws — *somewhere else entirely*, three functions later, with a stack trace that points at the symptom rather than the cause.

```ts
// Same lie, four other dialects:
const u = JSON.parse(raw) as User;
const u2 = data as unknown as User;                       // the "double assertion" — worse
function parse<T>(s: string): T { return JSON.parse(s); } // ← the fake generic (Lesson 08)
const token = localStorage.getItem("token")!;             // might be null
```

> **The single sentence to carry:** *"`as` is not a check — it's you telling the compiler to stop checking. Validation is code that runs."*

---

## 3. Parse, don't validate

```ts
// ❌ VALIDATE — returns a boolean; the value keeps its untrusted type
function isValidEmail(s: string): boolean { return /.+@.+/.test(s); }
if (isValidEmail(input)) sendTo(input);   // `input` is still just `string`
// Nothing stops someone calling sendTo(otherString) tomorrow.

// ✅ PARSE — returns a NARROWER type that carries the proof
function parseEmail(s: string): Result<Email, "invalid_email"> {
  return /.+@.+/.test(s) ? Ok(s as Email) : Err("invalid_email");
}
function sendTo(to: Email) {}     // ← cannot be called with an unvalidated string
```

The difference is **where the knowledge lives**. A validator leaves the knowledge in your head ("I checked this earlier, I think"); a parser puts it in the type, where the compiler enforces it for the rest of the program. This is the [Lesson 09](../03-type-level/09-illegal-states-unrepresentable.md) idea applied to I/O.

---

## 4. Zod: schema as the single source of truth

The key insight — and the reason schema libraries beat hand-written predicates:

```ts
import { z } from "zod";

// ONE declaration. The runtime check and the static type come from the same place.
export const PaymentSchema = z.object({
  id: z.string().regex(/^pi_[A-Za-z0-9]+$/),
  amount_minor: z.number().int().nonnegative(),
  currency: z.enum(["usd", "eur", "gbp", "inr", "jpy"]),
  status: z.enum(["requires_capture", "succeeded", "failed", "refunded"]),
  customer: z.string().nullable(),
  created_at: z.string().datetime(),
  metadata: z.record(z.string().max(500)).default({}),
});

export type Payment = z.infer<typeof PaymentSchema>;
//          ^? { id: string; amount_minor: number; currency: "usd" | ... ; ... }
```

**They cannot drift.** Add a field to the schema and the type updates. Change a type in the schema and every consumer's compile breaks. Compare with a hand-written predicate ([Lesson 07](../02-type-system/07-narrowing-and-control-flow.md)), where the interface and the check are two files that silently disagree the moment someone edits one.

### `parse` vs `safeParse`
```ts
const p = PaymentSchema.parse(data);            // throws ZodError on failure
const r = PaymentSchema.safeParse(data);        // { success: true, data } | { success: false, error }
if (!r.success) return Err(toApiError(r.error));
```
**`safeParse` at boundaries you expect to fail** (user input, third-party responses — you want to handle it). **`parse` where failure means a bug** (your own config at boot — you want a loud crash).

### The features that matter in real code

```ts
// Transform — parse into the shape you actually want, not the shape you received
const PaymentSchema = z.object({
  amount_minor: z.number().int(),
  currency: z.enum(["usd", "eur"]),
  created_at: z.string().datetime().transform(s => new Date(s)),   // string in, Date out
}).transform(p => ({
  money: { amountMinor: p.amount_minor, currency: p.currency },    // snake_case → your domain
  createdAt: p.created_at,
}));

type Payment = z.infer<typeof PaymentSchema>;      // { money: {...}; createdAt: Date }
type Wire = z.input<typeof PaymentSchema>;          // the ORIGINAL wire shape
```
`z.input` vs `z.infer` (a.k.a. `z.output`) is the pair to know: **input is what the wire sends, output is what your code holds.** That's the `Serialized<T>` idea from [Lesson 11](../03-type-level/11-mapped-and-template-literal-types.md), made runtime-real.

```ts
// Refine — cross-field rules a plain schema can't express
const DateRange = z.object({
  start: z.string().datetime(),
  end: z.string().datetime(),
}).refine(r => new Date(r.start) < new Date(r.end), {
  message: "start must be before end",
  path: ["end"],                          // ← attaches the error to the right field
});

// Brand — connect Zod to Lesson 09's branded types
const PaymentId = z.string().regex(/^pi_/).brand<"PaymentId">();
type PaymentId = z.infer<typeof PaymentId>;      // string & z.BRAND<"PaymentId">

// Discriminated unions — fast and correct, unlike a plain union
const PaymentSchema = z.discriminatedUnion("status", [
  z.object({ status: z.literal("requires_capture"), authorized_at: z.string() }),
  z.object({ status: z.literal("succeeded"), captured_at: z.string() }),
  z.object({ status: z.literal("failed"), failure_code: z.string() }),
]);
```
**Use `discriminatedUnion`, not `z.union`, whenever there's a discriminant.** `z.union` tries every member and reports a confusing aggregate error; `discriminatedUnion` jumps straight to the right branch and gives a precise error. It's both faster and better-diagnosing.

```ts
// Strictness — mirrors the API-side decision from API Lesson 11
z.object({ ... })                 // strips unknown keys (the default)
z.object({ ... }).strict()        // ERRORS on unknown keys  ← for REQUESTS you receive
z.object({ ... }).passthrough()   // keeps unknown keys
```
**Requests you receive: `.strict()`** — a typo'd field must be an error ([API L11](../../API/02-rest-design/11-versioning-and-evolution.md)). **Responses you consume: the default strip** — you must be a tolerant reader, or the server can never add a field. That asymmetry is exactly the API-track rule, now enforced in code.

---

## 5. The four boundaries, done properly

### Boundary 1 — Environment variables (do this first, in every project)
```ts
// src/config.ts
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, "JWT_SECRET must be at least 32 characters"),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
  WEBHOOK_SECRET: z.string().startsWith("whsec_"),
});

const parsed = EnvSchema.safeParse(process.env);
if (!parsed.success) {
  console.error("❌ Invalid environment:");
  for (const i of parsed.error.issues) console.error(`  ${i.path.join(".")}: ${i.message}`);
  process.exit(1);                     // ← crash at BOOT, not at first request
}
export const env = Object.freeze(parsed.data);
```
**This is the highest value-per-line code in any project.** Ten minutes of work converts "the service started and mysteriously 500s on the payments endpoint at 2am because `WEBHOOK_SECRET` was missing" into "it refused to start and told you which variable." Note `z.coerce.number()` — env vars are always strings.

### Boundary 2 — API responses
```ts
// src/lib/api.ts
export async function apiFetch<S extends z.ZodTypeAny>(
  path: string,
  schema: S,
  init?: RequestInit,
): Promise<Result<z.infer<S>, ApiError>> {
  const res = await fetch(`${env.API_URL}${path}`, {
    ...init,
    headers: { accept: "application/json", ...init?.headers },
    signal: AbortSignal.timeout(10_000),                    // API L18 — always a timeout
  });

  const requestId = res.headers.get("x-request-id") ?? undefined;

  if (!res.ok) {
    const problem = await res.json().catch(() => null);     // ← errors have a DIFFERENT shape
    const parsed = ProblemSchema.safeParse(problem);
    return Err(parsed.success
      ? new ApiError(res.status, parsed.data.code, parsed.data.detail, requestId)
      : new ApiError(res.status, "unknown_error", res.statusText, requestId));
  }

  const body: unknown = await res.json();                   // ← unknown, always
  const parsed = schema.safeParse(body);
  if (!parsed.success) {
    // A contract violation: the server sent something we don't understand.
    logger.error({ issues: parsed.error.issues, path, requestId }, "response_schema_mismatch");
    return Err(new ApiError(502, "invalid_response", "Server returned an unexpected shape", requestId));
  }
  return Ok(parsed.data);
}

const r = await apiFetch("/v1/payments/pi_1", PaymentSchema);
if (r.ok) r.value.money.amountMinor;     // ✅ fully typed AND verified
```

Three decisions in that function worth defending:
1. **Errors are parsed separately** — the error body is `problem+json`, not the success shape ([API L10](../../API/02-rest-design/10-errors-and-problem-details.md)).
2. **A schema mismatch is logged and becomes a 502**, not a crash. The server broke its contract; you degrade rather than white-screen.
3. **`x-request-id` is captured** so a user-facing error can carry it — the support workflow from [API Lesson 20](../../API/04-production/20-observability.md).

> **Should you validate responses in production?** The honest answer to give: *"yes for third-party APIs, always. For my own API, I'd validate in development and staging so contract drift is caught immediately, and consider sampling in production if payloads are large — but the cost is usually negligible next to the network round trip, and the diagnostic value when the contract breaks is high."* Nuance beats dogma here.

### Boundary 3 — Forms and user input
```ts
const CreatePaymentForm = z.object({
  amount: z.string()
    .regex(/^\d+(\.\d{1,2})?$/, "Enter a valid amount")
    .transform(s => Math.round(parseFloat(s) * 100))        // string → minor units (API L05)
    .refine(n => n >= 50, "Minimum charge is 0.50")
    .refine(n => n <= 99_999_999, "Amount too large"),
  currency: z.enum(["usd", "eur", "gbp", "inr"]),
  description: z.string().max(1000).optional(),
  customer: z.string().startsWith("cus_").optional(),
});

type FormInput = z.input<typeof CreatePaymentForm>;    // { amount: string; ... }  ← the <input> shape
type FormOutput = z.output<typeof CreatePaymentForm>;  // { amount: number; ... }  ← the API shape
```
**That input/output split is exactly why forms are painful without a schema library:** the DOM gives you strings, your API wants integers in minor units, and the transformation plus its validation belongs in one place. You'll wire this to React Hook Form in [Lesson 17](../05-ecosystem/17-typescript-with-react.md).

### Boundary 4 — `localStorage` and persisted state
```ts
export function readStored<S extends z.ZodTypeAny>(key: string, schema: S): z.infer<S> | null {
  try {
    const raw = localStorage.getItem(key);
    if (raw === null) return null;
    const parsed = schema.safeParse(JSON.parse(raw));
    if (!parsed.success) { localStorage.removeItem(key); return null; }   // stale/corrupt → discard
    return parsed.data;
  } catch { return null; }                  // quota errors, private mode, corrupt JSON
}
```
**`localStorage` is the boundary people most often forget**, and it's uniquely nasty because the data was written by an *older version of your own code*. Last month's shape is today's crash. Parsing turns a crash into a clean reset.

---

## 6. Deriving schemas from OpenAPI — closing the loop

This is where the API track and the TypeScript track meet properly.

```bash
npx openapi-typescript openapi.yaml -o src/generated/api.d.ts   # types
npx openapi-zod-client openapi.yaml -o src/generated/client.ts  # types + Zod schemas + client
```

```
openapi.yaml  ──┬──> server: express-openapi-validator (request + response validation)
                └──> client: generated types + Zod schemas + typed fetch client
```

**One contract, both sides, no drift.** A breaking change in the spec becomes a client compile error ([API L25](../../API/06-craft/25-openapi-contract-first.md)) — which is the whole point of contract-first development, and a genuinely strong thing to describe in an interview.

The reverse direction also exists and is worth knowing: define Zod schemas as the source of truth and generate OpenAPI from them (`@asteasolutions/zod-to-openapi`, or tRPC/Hono with OpenAPI output). Choose based on who owns the contract — **if consumers outside your repo depend on it, the spec should be the source; if it's a first-party app, schemas-first is faster.**

---

## 7. The alternatives

| Library | Bundle | Notable |
|---|---|---|
| **Zod** | ~12–14 kB | The default. Best ecosystem, best docs. Zod 4 is much faster than 3 |
| **Valibot** | ~1–3 kB | Modular/tree-shakeable — you import only the validators you use. **Best for client bundles** |
| **ArkType** | ~10 kB | Type-syntax schemas (`type({ id: "string" })`), very fast |
| **TypeBox** | ~5 kB | Produces **JSON Schema**, so it pairs naturally with OpenAPI/Fastify |
| **Yup** | ~20 kB | Older, weaker inference. Legacy choice |
| **class-validator** | — | Decorator-based; the NestJS default. Familiar from Spring's Bean Validation |

**The decision:** Zod on the server (bundle size doesn't matter, ecosystem does); **Valibot in the browser if bundle size is a live concern**; TypeBox if you need JSON Schema for OpenAPI. `class-validator` if you're in NestJS.

> Bundle awareness matters more than people expect: 14 kB of Zod in a client bundle is real, and if you only validate three shapes, Valibot's tree-shaking can cut that by 90%.

---

## 8. Production rules

| Rule | Why |
|---|---|
| **Every external input is `unknown` until parsed** | Types are claims; parsing is a check |
| **Never `as` external data. Never `parse<T>(s): T`** | Both are assertions wearing validation's clothes |
| **Derive types from schemas (`z.infer`), never hand-write both** | The only way they can't drift |
| **Validate env at boot and `process.exit(1)` on failure** | Fail at startup, not at 2am on one endpoint |
| **`safeParse` for expected failures; `parse` where failure is a bug** | Match the mechanism to the situation |
| **`.strict()` on requests you receive; default strip on responses you consume** | Strict in, tolerant out — the API-track asymmetry |
| **`discriminatedUnion` over `union` when there's a discriminant** | Faster, and vastly better error messages |
| **Parse `localStorage`; discard on mismatch** | Old versions of your own code wrote that data |
| **Map validation errors to your API error contract, with field paths** | [API L10](../../API/02-rest-design/10-errors-and-problem-details.md) — clients need `field` + `code` |
| **Keep schemas next to the domain type they describe** | One file, one concept |
| **Generate schemas from OpenAPI when the spec is the contract** | Closes the loop; breaking changes become compile errors |
| **Consider Valibot for client bundles** | Zod is ~14 kB you may not need |

---

## 9. Interview traps

**Q1. "Your API returns `{id: number}` but your type says `{id: string}`. When do you find out?"**
At runtime, far from the fetch, when something calls a string method on a number. `res.json()` is `any`, which assigns to anything silently. Fix: parse at the boundary with a schema and derive the type from it.

**Q2. "What's wrong with `JSON.parse(raw) as User`?"**
`as` disables checking — it's not validation, it's you overriding the compiler. Nothing ran. The error appears later, somewhere else, with a misleading stack trace.

**Q3. "Why Zod instead of a hand-written type guard?"**
One source of truth. With a guard you maintain an interface *and* a check, and they drift the moment someone edits one; `z.infer` makes drift impossible. You also get composition, transforms, precise per-field error paths, and cross-field `refine` rules for free.

**Q4. "Where do you put validation?"**
At every boundary: HTTP responses, request bodies, env vars, `localStorage`, URL params, form input, WebSocket messages, and third-party callbacks. **Not** between your own internal functions — inside the boundary, the compiler's guarantees hold and re-validating is noise.

**Q5. "Should you validate responses from your own API in production?"**
Nuance, not dogma: yes for third parties always; for your own API, validate in dev/staging to catch contract drift immediately, and keep it in production unless payloads are large enough for the cost to matter (it rarely is, next to the network). The diagnostic value when a contract breaks is high — you get a precise "response_schema_mismatch" log instead of a mystery.

**Q6. "`parse` vs `safeParse`?"**
`parse` throws — use it where failure means a programmer/config bug you want to crash on (env at boot). `safeParse` returns a result — use it where failure is expected and must be handled (user input, third-party responses). Same distinction as `Result` vs exceptions from [Lesson 09](../03-type-level/09-illegal-states-unrepresentable.md).

**Q7. "What's `z.input` vs `z.infer`?"**
With `.transform()`, the input type (what the wire/form sends) differs from the output type (what your code holds). `z.input` is the former, `z.infer`/`z.output` the latter. It's how you model "strings on the wire, `Date` and minor units internally" in one declaration.

**Q8. "How do you keep client and server types in sync?"**
Generate both from one contract: OpenAPI → types + Zod schemas + a typed client for the front end, and OpenAPI-based request/response validation on the server. A breaking spec change becomes a client compile error rather than a production incident.

**Q9. "Isn't validating everything slow?"**
Measure before assuming. Zod 4 parses a typical API object in microseconds — negligible next to a network round trip or a DB query. Where it *can* matter: very large arrays (validate a sample or use a faster library), and hot server paths (TypeBox compiles to fast validators). Bundle size in the browser is the more common real concern, which is Valibot's case.

**Q10. "How do you handle a schema mismatch in production?"**
Don't crash the request. Log it with the issues and the `request_id`, return a 502-class error to the caller, and alert — because a response-shape mismatch means the server broke its contract. Degrading beats white-screening, and the log tells you exactly which field changed.

---

## 10. Build & break

### Build — Ledger's validated boundary
1. **`src/config.ts`** — env schema, `safeParse`, `process.exit(1)` with per-field messages.
2. **`src/schemas/`** — `payment.ts`, `customer.ts`, `refund.ts`, `problem.ts`. Use `discriminatedUnion` for payment status, `.brand()` for IDs, `.transform()` for `Date` and money.
3. **`src/lib/api.ts`** — the `apiFetch` from §5, returning `Result`.
4. **`src/lib/storage.ts`** — schema-validated `localStorage` helpers.
5. **The error mapper** — `ZodError` → your RFC 9457 shape:
```ts
export function toValidationError(e: z.ZodError): ValidationError {
  return new ValidationError(e.issues.map(i => ({
    field: i.path.join("."),                    // "items.2.quantity" — matches API L10
    code: i.code,                                // "invalid_type", "too_small", …
    detail: i.message,
  })));
}
```
That field path format matches what you specified in the API track — so a client can highlight the exact input.

### Build — prove they can't drift
```ts
export const PaymentSchema = z.object({ /* ... */ });
export type Payment = z.infer<typeof PaymentSchema>;
```
Add a field to the schema and watch every consumer that destructures update. Then remove one and watch consumers break. **Then try the hand-written alternative** — a separate `interface` plus a predicate — and edit only one of them. Nothing breaks, and that silence is the bug.

### Break — six experiments
1. **The `as` lie.** `const u = JSON.parse('{"id":42}') as {id: string}; u.id.toUpperCase()`. Compiles, crashes. Replace with a schema and watch it fail *at the boundary* with a precise message.
2. **Missing env var.** Delete `DATABASE_URL` and start the server, first without validation (mysterious failure on the first DB call), then with (clear startup crash naming the variable).
3. **Stale `localStorage`.** Store `{theme: "dark"}`, then change the schema to require `{theme, density}`. Reload. Unparsed → crash; parsed → clean reset.
4. **Contract drift.** Change a field's type in your API mock and watch the client's schema catch it at the boundary with a named field, rather than three functions later.
5. **`union` vs `discriminatedUnion`.** Build the payment union both ways and feed it an invalid object. Compare the error messages — the `union` one is a wall of aggregated failures.
6. **Bundle cost.** Build a client bundle with Zod and with Valibot for the same three schemas. Compare the sizes.

### Explain out loud (90 seconds)
1. Where the boundary is — enumerate six places.
2. Why `as` isn't validation.
3. "Parse, don't validate" in one sentence.
4. Why `z.infer` beats a hand-written type + guard.
5. Strict-in / tolerant-out, and where each applies.

---

## What's next

You can get trustworthy data in. Next: what happens when getting it **fails** — typing async correctly, why `catch` gives you `unknown`, and how to thread `Result` through real code without drowning in it.

Next → **[Lesson 16: Async, errors & the `Result` pattern](16-async-and-errors.md)**
