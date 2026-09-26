# Lesson 22 — Code review & design judgement

> **Why this lesson exists:** knowing TypeScript and *applying judgement* with it are different skills, and the second is what gets you promoted. This lesson is the review checklist you run on your own work and anyone else's, the ten findings you'll make most often, and the design questions that don't have syntax answers. It doubles as a revision pass over the whole track.

**Time:** ~50 minutes, then it's a reference · **Prereq:** Modules 1–5

---

## 1. The idea in one sentence

> **Review in order of consequence: an unvalidated boundary outranks a missing type annotation, and a type that permits illegal states outranks a naming preference — so sweep the expensive things first.**

---

## 2. The review, in order

### Pass 1 — Escape hatches (2 min)
Grep first. These are where correctness was abandoned.
```bash
rg '\bany\b' --type ts -g '!*.test.ts' | wc -l
rg ' as [A-Z]' --type ts | grep -v 'as const'
rg '!\.' --type ts                     # non-null assertions
rg '@ts-ignore|@ts-expect-error' --type ts
```
```
[ ] Every `any` justified with a comment, or replaceable by `unknown`/a generic
[ ] Every `as` is either a branded constructor, a post-check narrowing, or a test
[ ] Every `!` has a comment naming the invariant
[ ] Zero `@ts-ignore`; every `@ts-expect-error` has a description and a ticket
[ ] No `function parse<T>(s: string): T` — the fake generic
```
**The single highest-value question in a TypeScript review:** *"what would break if this `as` were wrong?"* If the answer is "a runtime crash in production", it needs a real check.

### Pass 2 — Boundaries (3 min) — **the highest-consequence pass**
```
[ ] Every fetch/axios response parsed with a schema, not asserted
[ ] JSON.parse result typed `unknown` and parsed
[ ] Env validated at boot, with process.exit(1) on failure
[ ] localStorage/sessionStorage reads parsed, and discarded on mismatch
[ ] URL/query params parsed (they're strings, always)
[ ] Form input parsed, with z.input/z.output where transforms exist
[ ] catch variables narrowed, never assumed to be Error
[ ] Third-party SDK callbacks treated as unknown
```
A missing boundary check is the finding that actually causes incidents. Everything below is cheaper to fix later.

### Pass 3 — Modelling (3 min)
```
[ ] Count representable vs meaningful states — is there a gap?
[ ] Multi-state concepts are discriminated unions, not optional-field soup
[ ] A `return null` for an "impossible" branch → the type is too wide
[ ] IDs, money, units and validated strings are branded
[ ] Money carries its currency in the type
[ ] Enums are string literal unions (or `as const`), not `enum`
[ ] Exhaustive switches have `assertNever` in the default
[ ] Input types Omit server-owned fields (id, status, createdAt) — mass-assignment defence
```

### Pass 4 — Signatures (2 min)
```
[ ] Exported functions annotate their return type
[ ] Generic parameters appear at least twice (else they're noise or a lie)
[ ] Parameters accept `readonly T[]` unless mutation is intended
[ ] Function-property syntax over method shorthand in your own interfaces
[ ] Overloads justified — union/generic/conditional tried first
[ ] Async functions return `Promise<T>` with `T` annotated; errors typed if using Result
```

### Pass 5 — Config (1 min)
```
[ ] strict: true
[ ] noUncheckedIndexedAccess: true
[ ] exactOptionalPropertyTypes: true
[ ] skipLibCheck: true (app) / false in one CI job (library)
[ ] tsc --noEmit is a required CI check
[ ] Type-aware ESLint: no-floating-promises, no-misused-promises
[ ] TypeScript version pinned; editor set to the workspace version
```

### Pass 6 — Complexity (1 min)
```
[ ] Can every non-trivial type be explained in 60 seconds?
[ ] Complex types live in shared utils/library code, not feature files
[ ] Non-trivial types have .test-d.ts tests
[ ] Public types wrapped in Prettify for readable hovers
[ ] No type is doing work a simpler domain model would remove
```

---

## 3. The ten findings you'll make most often

In rough order of frequency:

| # | Finding | Severity | Why it's common |
|---|---|---|---|
| 1 | Unvalidated API response (`as User`, or bare `res.json()`) | 🔴 | It looks typed and isn't |
| 2 | `any` from an untyped dependency, spreading silently | 🔴 | Arrives implicitly; nobody typed the word |
| 3 | `!` used to silence an error nobody understood | 🟠 | Fastest way to make the squiggle go away |
| 4 | Optional-field soup instead of a discriminated union | 🟠 | It's the shape the data "naturally" has |
| 5 | Unbranded IDs — `getPayment(customerId)` compiles | 🟠 | Structural typing hides it |
| 6 | Non-exhaustive switch with no `assertNever` | 🟠 | Works today; breaks silently when a variant is added |
| 7 | `enum` where a literal union belongs | 🟡 | Familiarity from other languages |
| 8 | Exported functions with inferred return types | 🟡 | Invisible cost until the build is slow |
| 9 | `Record<string, T>` without `noUncheckedIndexedAccess` | 🟡 | The type lies about presence |
| 10 | A clever type in a feature file | 🟡 | Written when the author was enjoying themselves |

**If you check only these ten, you'll catch most of what matters** — and #1 and #2 alone account for the majority of "but it type-checked!" incidents.

### The severity scale
| Level | Meaning |
|---|---|
| **🔴 Blocker** | Ships a runtime crash or a silent wrong value. Unvalidated boundary, `any` on external data, a lying type predicate |
| **🟠 Must fix** | Permits illegal states or hides real bugs. Optional-field soup, unbranded IDs, unexplained `!` |
| **🟡 Should fix** | Real cost, fixable later. Missing return annotations, `enum`, over-clever types |
| **🔵 Nit** | Preference. `interface` vs `type` given consistency, naming, formatting |

**Lead with blockers, and mark nits as nits.** A review that buries a missing boundary check under six naming suggestions has failed.

### How to phrase a finding
```
❌ "Don't use `as` here."
✅ "🔴 `res.json() as Payment` doesn't validate anything — if the API changes
    `amount_minor` to a string, this compiles and crashes in `formatMoney`
    three calls later. Suggest `PaymentSchema.safeParse(await res.json())`,
    which also gives you the field path in the error. TS Lesson 15 has the pattern."
```
**Severity → what → the concrete consequence → the fix.** The consequence is what gets it fixed; without it you're asserting taste.

---

## 4. The design questions (no syntax answers)

These are what senior TypeScript judgement actually consists of.

### "How strict should our types be?"
Strict where correctness matters; pragmatic where it doesn't.
```ts
// ✅ Strict — money, IDs, state. Getting it wrong is expensive.
type Money<C extends Currency> = { amountMinor: number; currency: C };

// ✅ Pragmatic — analytics metadata. Getting it wrong costs a log line.
type EventProps = Record<string, string | number | boolean>;
```
The test: **what does being wrong cost?** Uniform strictness everywhere is as much a mistake as uniform looseness — it spends your complexity budget where it buys nothing.

### "Should this be generic?"
Does the type parameter appear twice? Is there more than one call site? Would a union be clearer? **Most "should this be generic" answers are no.**

### "Types or runtime validation?"
Both, at different places. Types inside the boundary, validation at it. **Never validate between your own internal functions** — inside the boundary the compiler's guarantees hold and re-checking is noise that implies you don't trust your own types.

### "How do we handle an API that might change?"
Open enums with `LiteralUnion`, tolerant response parsing (strip unknowns), exhaustive switches with a `default` branch, and schema-derived types so a contract change is a compile error rather than a production one. That's [API Lesson 11](../../API/02-rest-design/11-versioning-and-evolution.md) and [Lesson 15](../04-real-code/15-the-boundary-and-parsing.md) combined.

### "When do we reach for a library?"
Schema validation: always (Zod/Valibot). Utility types: `type-fest` for deep/recursive ones. State management: when prop drilling hurts. **Type-level gymnastics: when you can't enumerate the edge cases yourself.**

### "How much should types encode business rules?"
```ts
// ✅ Structural rules — cheap and permanent
type Capturable = Extract<Payment, { status: "requires_capture" }>;

// ❌ Dynamic rules — types can't see runtime values
type ValidAmount = number;   // "must be under the merchant's daily limit" isn't expressible
```
**Types encode what's knowable at compile time.** Runtime rules need runtime checks — and trying to force them into the type system produces unreadable types that are still wrong.

---

## 5. Review scenarios

**"This PR adds `as any` in three places."**
Ask *why* at each. Usually: an untyped dependency (→ an adapter module with one contained `any` and validation), an unvalidated API response (→ a schema), or a type the author couldn't work out (→ pair on it; it's often a genuine modelling problem). **Blocking on `any` without offering the alternative is how reviewers get ignored.**

**"This type is 40 lines of conditional types."**
Two questions: *"Can you explain it in 60 seconds?"* and *"Is this library code or feature code?"* In a shared util, complexity amortises across consumers. In a feature file it has one beneficiary and a permanent maintenance cost — and it's usually a signal the domain model is wrong.

**"Everything is `Partial<T>`."**
`Partial` is right for "any subset may be provided" and wrong for "these specific fields, optionally". It also can't express `null`-means-clear. Usually the author wanted an explicit patch type.

**"The team wants to disable `strictNullChecks` to move faster."**
Push back with the specific cost: it's the flag that catches the most common runtime error in JavaScript, and re-enabling it later costs 10× more than keeping it. Then offer the real compromise — per-directory strictness with a ratchet ([Lesson 19](../05-ecosystem/19-migration-and-strictness.md)) — so nobody is blocked while the guarantee is preserved where it matters.

**"A junior wrote `function getData<T>(url: string): Promise<T>`."**
A teaching moment, not a rejection. Explain that it's `as` in disguise — the caller invents `T`, nothing validates it, and the error surfaces far from the fetch. Show the schema version. **This is the most common TypeScript misconception**, and it looks like good practice, which is exactly why it needs explaining rather than flagging.

**"Should we adopt tRPC / GraphQL codegen / OpenAPI codegen?"**
Depends on who owns the contract. Same repo, TS both ends, shipped together → tRPC. External consumers or a published API → OpenAPI as the source of truth, generate both sides. GraphQL → its codegen is mature and worth it. **The principle: one contract, both sides generated, so drift becomes a compile error** ([API Lesson 25](../../API/06-craft/25-openapi-contract-first.md)).

---

## 6. Your own pre-merge gate

```
BEFORE I open the PR:
[ ] No new `any`, `as`, or `!` without a comment
[ ] Every new external input is parsed, not asserted
[ ] New unions have exhaustive handling with assertNever
[ ] New exported functions annotate their return type
[ ] New non-trivial types have .test-d.ts tests
[ ] tsc --noEmit and lint pass locally
[ ] I hovered the key types and they're what I intended
[ ] I can explain every type I wrote in 60 seconds
```

That last item is the one that catches over-engineering, and it's worth taking literally.

---

## 7. Interview traps

**Q1. "Review this TypeScript."**
Announce the method first: *"I'll look at escape hatches, then boundaries, then whether the model permits illegal states, then signatures, then config, then complexity. Boundaries are where I'll spend most time, because that's where the compiler's guarantees stop."* **Announcing an ordered method is most of the score.**

**Q2. "What's the first thing you look for?"**
`any` and `as` on external data. That's where the compiler was told to stop checking, and it's where "but it type-checked!" incidents come from.

**Q3. "When is `any` acceptable?"**
Genuinely dynamic metaprogramming, a third-party type you can't fix (contained in one adapter module), and migration placeholders — always commented. **Prefer `unknown`**: it's the only one where the compiler keeps helping.

**Q4. "How do you stop a team from over-engineering types?"**
A written rule everyone can apply: *"if you can't explain it in 60 seconds, simplify it or move it to a shared util with tests."* Plus review complexity as a first-class finding, not a nit. And notice that unreadable types are usually a symptom — the domain model is wrong.

**Q5. "Types or tests?"**
Both, for different failures. Types eliminate whole *categories* — wrong shape, missing case, illegal state. Tests verify *behaviour*. Good types make tests shorter, because you stop testing what the compiler proves.

**Q6. "How do you review a type you don't understand?"**
Don't wave it through. Paste it into the Playground, instantiate it with a small example, and hover the result. If you can't follow it in five minutes, that's itself the finding — **maintainability is a review criterion**, and "the author is the only person who can change this" is a real risk.

**Q7. "What does TypeScript not fix?"**
Wrong logic, bad domain models, unvalidated inputs, missing error handling, race conditions, performance. **It stops you saying the wrong thing; it doesn't stop you meaning the wrong thing.** A well-typed program can be entirely incorrect.

**Q8. "How would you introduce TypeScript standards to a team with none?"**
Not a 40-page document. Start with three enforceable rules: `strict: true` in CI, no `@ts-ignore`, and all external data parsed. Enforce mechanically (tsconfig + ESLint + CI) rather than socially, and add rules only when a real bug motivates one. **Rules nobody can enforce are decoration.**

---

## 8. Build & break

### Build — review something real
Pick an open-source TypeScript project you don't know. Run the six passes and write graded findings. You'll find real issues in well-regarded codebases — that's the calibration, and it teaches you which trade-offs are deliberate.

### Build — review Ledger
Run the checklist on your own shared type layer and API client. Fix every 🔴 and 🟠. Write `docs/typescript-conventions.md`:
```markdown
## When `any` is acceptable
- Contained inside an adapter module for an untyped dependency, with validation at the exit
- Never on external data. Never in a function signature we own.

## When `as` is acceptable
- Inside a validating branded-type constructor (the one audited place)
- After a check the compiler can't see, with a comment
- In tests

## Boundaries — all of these MUST be parsed
fetch responses · JSON.parse · localStorage · process.env · URL params · forms
· WebSocket messages · third-party callbacks · catch variables

## Modelling
- Multi-state concepts are discriminated unions
- IDs, money and units are branded
- Exhaustive switches end with assertNever
- Input types Omit server-owned fields
```
**A conventions doc in the repo is a genuinely senior artifact** — it converts a reviewer's preferences into the team's rules.

### Explain out loud (90 seconds)
1. The six review passes, in order, and why that order.
2. The first thing you grep for and why.
3. Three of the ten common findings with their consequences.
4. Your rule for when a type is too complex.
5. What TypeScript doesn't fix.

---

## The TypeScript track is complete

22 lessons: foundations, the type system, type-level programming, real code, ecosystem, interview.

**Before moving to React, do these four things:**

1. **Retake the 15-question self-test** from the [README](../README.md), cold and timed. Compare with your baseline.
2. **Build the shared type layer** — brands, `Money`, the `Payment` union, `Result`, the schemas, the typed API client. It's what the React track consumes, and it's the portfolio artifact.
3. **Tier 1 and 2 type challenges** until Tier 1 is under two minutes each.
4. **Turn on `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`** in a real project and fix the fallout. That fallout is the lesson.

Then → **[React: Lesson 01](../../React/README.md)**

> The React track assumes everything here. Components are typed APIs, state is a discriminated union, server data is parsed at the boundary, and every `switch` is exhaustive. You've already built the hard half.
