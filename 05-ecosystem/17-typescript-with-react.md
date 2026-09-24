# Lesson 17 — TypeScript with React

> **Why this lesson exists:** React is where most TypeScript is written, and where most `any` gets added out of frustration. The patterns are finite — component props, hooks, events, generics, context, polymorphic components — and once you know them you stop fighting. This lesson is also the **bridge into the React track**: everything here is assumed there.

**Time:** ~70 minutes · **Prereq:** Modules 1–4 · Basic React familiarity

---

## 1. The idea in one sentence

> **A component's props are its API — so type them the way you'd design an API contract: precise, minimal, and impossible to call incorrectly.**

That's [API Lesson 01](../../API/01-foundations/01-what-an-api-really-is.md) at component scale, and it's the frame that makes every decision below obvious.

---

## 2. Typing components

```tsx
// ✅ The modern default: a plain function with typed props
type ButtonProps = {
  children: React.ReactNode;
  variant?: "primary" | "secondary" | "danger";
  onClick?: () => void;
  disabled?: boolean;
};

function Button({ children, variant = "primary", onClick, disabled }: ButtonProps) {
  return <button className={variant} onClick={onClick} disabled={disabled}>{children}</button>;
}
```

### Why not `React.FC`?
```tsx
const Button: React.FC<ButtonProps> = ({ children }) => <button>{children}</button>;
```
Historically `React.FC` implicitly added `children`, which meant components that took no children still accepted them. **React 18's types removed that**, so the main objection is gone. What remains:
- It makes **generic components awkward** (`const List: React.FC<Props<T>>` can't introduce `T`).
- It adds nothing a plain function signature doesn't already give you.
- It's an extra indirection when reading the code.

> **The answer to give:** *"`React.FC` was avoided because of the implicit `children` — that's fixed in React 18+. I still use a plain function because it's simpler and it doesn't get in the way of generic components. Either is fine now; the old blanket 'never use FC' advice is out of date."* Knowing the history, not just the rule, is the signal.

### `children` and the `ReactNode` family
```tsx
type Props = {
  children: React.ReactNode;            // ✅ anything renderable — the default choice
  icon: React.ReactElement;             // exactly one element
  render: (item: T) => React.ReactNode; // render prop
  component: React.ComponentType<P>;    // a component itself, not an element
};
```
| Type | Accepts |
|---|---|
| `ReactNode` | elements, strings, numbers, `null`, `undefined`, booleans, arrays — **use this for `children`** |
| `ReactElement` | a single JSX element only (no strings) |
| `JSX.Element` | ≈ `ReactElement`; prefer `ReactElement` |
| `ComponentType<P>` | a function or class component — for `as`/`component` props |

### Extending HTML element props — the pattern you'll use most
```tsx
// A button that accepts every native button attribute plus your own
type ButtonProps = React.ComponentPropsWithoutRef<"button"> & {
  variant?: "primary" | "secondary";
  loading?: boolean;
};

function Button({ variant = "primary", loading, disabled, ...rest }: ButtonProps) {
  return <button {...rest} disabled={disabled || loading} data-variant={variant} />;
}
// <Button onClick={...} type="submit" aria-label="Save" formAction={...} />  ✅ all native props work
```
**`ComponentPropsWithoutRef<"button">` is the right helper** — better than the older `ButtonHTMLAttributes<HTMLButtonElement>` because it picks up everything React accepts for that element, and the `WithoutRef` variant avoids a `ref` conflict when you're not forwarding one.

```tsx
// Overriding a native prop's type — you must Omit first
type InputProps = Omit<React.ComponentPropsWithoutRef<"input">, "onChange"> & {
  onChange: (value: string) => void;       // your simpler signature, not the event
};
```

### `ref` — and the React 19 change
```tsx
// React 19+: ref is just a prop. No forwardRef needed.
type InputProps = React.ComponentPropsWithRef<"input"> & { label: string };
function Input({ label, ref, ...rest }: InputProps) {
  return <label>{label}<input ref={ref} {...rest} /></label>;
}

// React 18 and earlier:
const Input = React.forwardRef<HTMLInputElement, InputProps>(({ label, ...rest }, ref) => (
  <label>{label}<input ref={ref} {...rest} /></label>
));
Input.displayName = "Input";      // forwardRef loses the inferred name
```

---

## 3. Typing hooks

```tsx
// useState — inference usually suffices
const [count, setCount] = useState(0);                         // number
const [user, setUser] = useState<User | null>(null);           // ← annotate when the initial is null
const [items, setItems] = useState<Payment[]>([]);             // ← annotate empty arrays (else never[])
const [status, setStatus] = useState<"idle" | "loading">("idle");  // ← annotate to avoid widening

// useRef — three distinct forms, and they're not interchangeable
const inputRef = useRef<HTMLInputElement>(null);        // DOM ref: RefObject, .current readonly-ish
const timerRef = useRef<number | undefined>(undefined); // mutable box: MutableRefObject
const countRef = useRef(0);                              // mutable, inferred number

// useReducer — annotate the reducer, get everything free
type Action =
  | { type: "loading" }
  | { type: "loaded"; payments: Payment[] }
  | { type: "failed"; error: ApiError };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "loading": return { status: "loading" };
    case "loaded":  return { status: "success", payments: action.payments };
    case "failed":  return { status: "error", error: action.error };
    default:        return assertNever(action);         // ← exhaustive
  }
}
const [state, dispatch] = useReducer(reducer, { status: "idle" });
dispatch({ type: "loaded", payments });        // ✅ checked
dispatch({ type: "loaded" });                   // ❌ missing payments
```

**The `useReducer` + discriminated-union pattern is the single best TypeScript/React combination.** Actions are checked at every dispatch site, the reducer is exhaustive, and adding an action is a compile error until it's handled. It's [Lesson 09](../03-type-level/09-illegal-states-unrepresentable.md) applied to UI state.

### `useState` gotchas
```tsx
const [items, setItems] = useState([]);           // ❌ never[] — you can never add anything
const [items2, setItems2] = useState<Payment[]>([]);   // ✅

// Functional updates preserve the type correctly
setCount(c => c + 1);

// Storing a function in state requires the functional form (or it's called!)
const [fn, setFn] = useState<() => void>(() => () => console.log("hi"));
setFn(() => newFn);                                // ← the outer arrow is the updater
```

### Custom hooks — return a tuple or an object, deliberately
```tsx
// ❌ Returns (boolean | (() => void))[] — destructuring loses the types
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  return [on, () => setOn(o => !o)];
}

// ✅ `as const` makes it a tuple
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  return [on, toggle] as const;                   // readonly [boolean, () => void]
}

// ✅ Or an object — better for 3+ values, because names beat positions
function usePayment(id: PaymentId) {
  // ...
  return { payment, isLoading, error, refetch };
}
```
**Tuple for 1–2 values (so callers can rename), object for 3+** (so callers don't have to remember the order). That's the same API-design judgement as positional vs named parameters.

### Typed context — with the "not provided" guard
```tsx
type AuthContextValue = {
  principal: Principal | null;
  login: (email: Email, password: string) => Promise<Result<Principal, ApiError>>;
  logout: () => void;
};

// The trick: undefined default + a hook that throws. Consumers never handle undefined.
const AuthContext = React.createContext<AuthContextValue | undefined>(undefined);

export function useAuth(): AuthContextValue {
  const ctx = React.useContext(AuthContext);
  if (ctx === undefined) {
    throw new Error("useAuth must be used within <AuthProvider>");
  }
  return ctx;                                      // ✅ narrowed — never undefined
}
```
**This pattern is worth memorising.** The alternative — a fake default object — means a missing provider silently renders broken UI instead of failing loudly.

---

## 4. Typing events

```tsx
// Let contextual typing do the work — annotate the variable, not the parameter (Lesson 05)
<button onClick={e => e.currentTarget.disabled = true} />          // e inferred ✅
<input onChange={e => setValue(e.target.value)} />                 // e inferred ✅

// When you extract a handler, annotate it:
const handleClick: React.MouseEventHandler<HTMLButtonElement> = e => { /* ... */ };
const handleChange: React.ChangeEventHandler<HTMLInputElement> = e => setValue(e.target.value);
const handleSubmit: React.FormEventHandler<HTMLFormElement> = e => { e.preventDefault(); };

// Or annotate the parameter:
function onKeyDown(e: React.KeyboardEvent<HTMLInputElement>) {
  if (e.key === "Enter") submit();
}
```

**`target` vs `currentTarget` — a real distinction:**
- `currentTarget` is the element the handler is **attached to** — always correctly typed.
- `target` is what was actually clicked, which may be a child — typed as `EventTarget`, so it needs narrowing.

```tsx
<div onClick={e => {
  e.currentTarget;                     // HTMLDivElement ✅
  e.target;                             // EventTarget — could be any descendant
  if (e.target instanceof HTMLButtonElement) e.target.disabled = true;   // narrow it
}} />
```
`ChangeEvent<HTMLInputElement>` is the exception: React types its `target` precisely, which is why `e.target.value` works.

---

## 5. Generic components

```tsx
// A generic list — note the arrow-function comma hack in .tsx files
function List<T>({ items, renderItem, keyOf }: {
  items: readonly T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyOf: (item: T) => string;
}) {
  return <ul>{items.map((item, i) => <li key={keyOf(item)}>{renderItem(item, i)}</li>)}</ul>;
}

<List
  items={payments}
  keyOf={p => p.id}
  renderItem={p => <PaymentRow payment={p} />}     // p inferred as Payment ✅
/>
```

```tsx
// In a .tsx file, <T> is parsed as JSX. Two workarounds:
const List = <T,>(props: Props<T>) => { /* ... */ };        // trailing comma
const List2 = <T extends unknown>(props: Props<T>) => { };   // or a constraint
```

### A typed data table — the pattern behind every real table component
```tsx
type Column<T> = {
  [K in keyof T]: {
    key: K;
    header: string;
    render?: (value: T[K], row: T) => React.ReactNode;   // ← value typed per column!
  };
}[keyof T];

function DataTable<T extends { id: string }>({ rows, columns }: {
  rows: readonly T[];
  columns: readonly Column<T>[];
}) { /* ... */ }

<DataTable
  rows={payments}
  columns={[
    { key: "id", header: "ID" },
    { key: "money", header: "Amount", render: m => formatMoney(m) },   // m: Money ✅
    { key: "status", header: "Status", render: s => <Badge status={s} /> },  // s: PaymentStatus ✅
    { key: "nope", header: "X" },                                       // ❌ not a key of Payment
  ]}
/>
```
That `Column<T>` is the **map-then-index idiom** from [Lesson 11](../03-type-level/11-mapped-and-template-literal-types.md), and it's what makes `render`'s parameter correctly typed *per column*. This is the single most impressive small piece of TypeScript you can show in a React interview.

### Polymorphic components (`as` prop)
```tsx
type PolymorphicProps<E extends React.ElementType, P> = P & {
  as?: E;
} & Omit<React.ComponentPropsWithoutRef<E>, keyof P | "as">;

function Text<E extends React.ElementType = "span">(
  { as, ...rest }: PolymorphicProps<E, { size?: "sm" | "lg" }>
) {
  const Component = as ?? "span";
  return <Component {...rest} />;
}

<Text>hi</Text>                          // span props
<Text as="a" href="/x">link</Text>       // ✅ href allowed because it's an anchor
<Text as="button" href="/x" />           // ❌ href isn't a button prop
```
Genuinely useful for a design system — and a genuine complexity cost. **Use it in a shared component library; don't build it for one component.**

### Mutually exclusive props
```tsx
type XOR<A, B> =
  | (A & { [K in Exclude<keyof B, keyof A>]?: never })
  | (B & { [K in Exclude<keyof A, keyof B>]?: never });

type ActionProps = XOR<{ href: string }, { onClick: () => void }>;

<Action href="/x" />                     // ✅
<Action onClick={fn} />                  // ✅
<Action href="/x" onClick={fn} />        // ❌ can't be both
<Action />                                // ❌ must be one
```
A link-or-button component is the canonical case, and `XOR` ([Lesson 11](../03-type-level/11-mapped-and-template-literal-types.md)) beats a runtime `if` with a `!`.

---

## 6. Forms, async state, and the Ledger types

```tsx
// React Hook Form + Zod — the standard combination, with input/output types (Lesson 15)
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";

const CreatePaymentSchema = z.object({
  amount: z.string().regex(/^\d+(\.\d{1,2})?$/).transform(s => Math.round(+s * 100)),
  currency: z.enum(["usd", "eur", "gbp", "inr"]),
  description: z.string().max(1000).optional(),
});
type FormInput = z.input<typeof CreatePaymentSchema>;    // { amount: string; ... }  ← the <input>s
type FormOutput = z.output<typeof CreatePaymentSchema>;  // { amount: number; ... }  ← the API body

function PaymentForm({ onCreated }: { onCreated: (p: Payment) => void }) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<FormInput, unknown, FormOutput>({ resolver: zodResolver(CreatePaymentSchema) });

  const onSubmit = handleSubmit(async data => {       // data: FormOutput — already transformed ✅
    const r = await api.createPayment(data, crypto.randomUUID() as IdempotencyKey);
    if (r.ok) onCreated(r.value);
  });

  return (
    <form onSubmit={onSubmit}>
      <input {...register("amount")} />
      {errors.amount && <span>{errors.amount.message}</span>}
      <button disabled={isSubmitting}>Create</button>
    </form>
  );
}
```
The three-parameter `useForm<In, Ctx, Out>` is the piece people miss — it's what lets `data` arrive already transformed.

```tsx
// Async state as a discriminated union (Lesson 09), rendered exhaustively
function PaymentView({ id }: { id: PaymentId }) {
  const query = usePayment(id);     // returns Async<Payment, ApiError>
  switch (query.state) {
    case "idle":
    case "loading":  return <Skeleton />;
    case "error":    return <ErrorView error={query.error} retryable={query.error.retryable} />;
    case "success":  return <PaymentDetail payment={query.data} />;   // data guaranteed ✅
    default:         return assertNever(query);
  }
}
```

---

## 7. Production rules

| Rule | Why |
|---|---|
| **Props are an API — design them, don't accumulate them** | Same discipline as endpoint design |
| **Extend `ComponentPropsWithoutRef<"tag">` for wrappers** | Consumers get every native attribute for free |
| **`Omit` before overriding a native prop's type** | Otherwise you get an intersection that's `never` |
| **Annotate `useState` for `null`, empty arrays, and literal unions** | Otherwise `never[]` or an unwanted widening |
| **`useReducer` + discriminated-union actions + `assertNever`** | Every dispatch checked; adding an action is a compile error |
| **Context: `undefined` default + a hook that throws** | A missing provider fails loudly instead of rendering broken UI |
| **`as const` on tuple returns from custom hooks** | Or destructuring gives a union array |
| **Tuple for 1–2 returns, object for 3+** | Renameable vs order-independent |
| **Annotate handler variables, not event parameters** | Contextual typing does the rest |
| **`currentTarget` over `target` unless you narrow** | `target` is `EventTarget` and may be a child |
| **`XOR` for mutually-exclusive props** | Compile-time instead of a runtime `!` |
| **`z.input`/`z.output` for forms** | Strings in, domain values out, one declaration |
| **Never `any` a prop — use `unknown` + narrowing, or a generic** | `any` on a prop spreads into the whole tree |

---

## 8. Interview traps

**Q1. "How do you type a React component?"**
A plain function with a props type. Mention the `React.FC` history accurately: it used to add implicit `children`, that's fixed in React 18+, and the remaining reason to avoid it is generic components. Then: extend `ComponentPropsWithoutRef<"button">` so consumers get native attributes.

**Q2. "Why does `useState([])` cause problems?"**
It infers `never[]`, so you can never add an element. Annotate: `useState<Payment[]>([])`. Same class of problem with `useState(null)`.

**Q3. "How do you type a context so consumers don't handle `undefined`?"**
Default the context to `undefined`, then export a hook that throws if it's missing. The throw narrows the type, so every consumer gets a non-nullable value and a missing provider fails loudly at the first render rather than silently.

**Q4. "Why does destructuring my custom hook's return lose types?"**
Because an array return is inferred as `(A | B)[]`, not a tuple. `as const` fixes it. Or return an object, which is better for three or more values.

**Q5. "`e.target` vs `e.currentTarget`?"**
`currentTarget` is the element the handler is attached to — precisely typed. `target` is whatever was actually hit, typed as `EventTarget`, so it needs `instanceof` narrowing. `ChangeEvent<HTMLInputElement>` is the special case where React types `target` for you.

**Q6. "How do you write a generic component in a `.tsx` file?"**
`<T,>` with a trailing comma, or `<T extends unknown>` — because bare `<T>` parses as JSX. Then a concrete example: a `List<T>` whose `renderItem` gets the correct item type.

**Q7. "Type a table where each column's `render` receives the correct cell type."**
The map-then-index union: `{ [K in keyof T]: { key: K; render?: (v: T[K]) => ReactNode } }[keyof T]`. Being able to produce this on a whiteboard is a strong signal — it's mapped types, key remapping and unions combined for a real purpose.

**Q8. "How do you make two props mutually exclusive?"**
`XOR<A, B>` — each branch intersected with the other's keys as optional `never`. The link-or-button component is the canonical case.

**Q9. "How do you type a form where inputs are strings but the API wants numbers?"**
A Zod schema with `.transform()`, then `z.input` for the form state and `z.output` for the submit handler — `useForm<In, Ctx, Out>` with `zodResolver`. One declaration covers validation, the input shape and the output shape.

**Q10. "Where does `any` sneak into React codebases?"**
Event handlers extracted without a type; `useState` with no annotation on `null`/`[]`; untyped third-party components; `props: any` on a wrapper; and `as` on `e.target`. The fix for most is contextual typing — annotate the variable and let inference do the rest.

---

## 9. Build & break

### Build — Ledger Console's typed component layer
1. **`Button`** extending `ComponentPropsWithoutRef<"button">` with `variant` and `loading`.
2. **`Action`** using `XOR<{href}, {onClick}>`.
3. **`DataTable<T>`** with the per-column `render` typing from §5.
4. **`AuthContext`** + `useAuth` with the throwing hook.
5. **`usePayments`** returning `Async<Payment[], ApiError>`, consumed with an exhaustive switch.
6. **`PaymentForm`** with `z.input`/`z.output` and RHF.

### Break — six experiments
1. `useState([])` then `setItems([payment])`. Read the `never[]` error.
2. Return `[value, setter]` from a custom hook without `as const`, destructure, and try to call the setter.
3. Use a context with a fake default object, forget the provider, and watch the UI render silently broken. Switch to the throwing hook.
4. Write `<T>(props) => ...` in a `.tsx` file and read the JSX parse error.
5. Add an action to a reducer's union without handling it — confirm `assertNever` catches it. Then remove `assertNever` and watch it compile silently.
6. Override a native prop (`onChange`) without `Omit` first and read the resulting `never` type.

### Explain out loud (90 seconds)
1. How you type a component's props, and the `React.FC` history.
2. The context + throwing hook pattern and what it prevents.
3. Why `as const` matters for hook returns.
4. The generic table column type, and which techniques it combines.
5. Where `any` usually sneaks in.

---

## What's next

You can type a React app. Next: making it **build fast** — project references, monorepos, and diagnosing a compiler that's gone slow, which is a real problem at scale and a question you'll be asked if you claim TypeScript experience.

Next → **[Lesson 18: Build tooling, monorepos & compiler performance](18-tooling-and-performance.md)**
