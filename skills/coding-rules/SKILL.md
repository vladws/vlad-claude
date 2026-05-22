---
name: coding-rules
description: Opinionated mandatory coding rules for TypeScript/React development. Contains non-negotiable standards covering code structure, code co-location, React patterns, TypeScript safety, GraphQL, testing, i18n, and tooling. Consult this skill whenever writing or modifying any TypeScript or React code — components, hooks, GraphQL queries, tests, i18n strings — even if the user hasn't explicitly asked about coding standards. Also use when reviewing code, refactoring, or when something "looks off." These rules override default coding instincts.
---

# Coding Rules

These rules override your default coding instincts. The patterns you reach for by reflex — adding comments to explain code, extracting one-line helpers, sprinkling `useCallback`, destructuring hook returns — are the exact things to suppress here. The pre-flight checklist below is injected into context before every code-mutating tool call; the full rules and rationale follow it. Auditing the resulting diff against these rules is handled by `/self-review` and `/engage`.

---

## Pre-flight checklist — every rule in brief

Each bullet is the terse form of a rule. The full rationale and examples for each follow in the sections below. This list is exhaustive — every rule in this document is represented here.

<!-- BEGIN PREFLIGHT -->
**Code structure**
- **No indirection** — no one-line helpers, aliased variables, or wrapper functions called once. Inline at the call site.
- **Inline single-use `Props`/param types** — extract a type alias only if reused in 2+ places. Top-level/exported components are not exempt.
- **No nested ternaries** — one ternary is fine; 2+ levels become a named pure function above the component.
- **Positive conditions first** in `if/else` and 2-branch ternaries. Only early-return guards (`if (!x) return`) are exempt.
- **Named-param objects** for any regular function (not a component or hook) with 2+ params.
- **Early returns over nested conditions** — validate preconditions at the top; keep the happy path at the lowest indent.
- **No `switch`** — use `if` with early returns.
- **No IIFEs in components** — extract `(() => {...})()` as a named pure function above.
- **Curly braces on every block** — wrap all `if`/`else`/`for`/`while` bodies in `{}`, even single-liners.

**React**
- **No `useCallback` / `useMemo`** unless there's a measured perf problem.
- **No destructuring hook returns or props** — use the whole object (`result.data`, `props.nav`). Tuple hooks (`useState`, `useReducer`) excepted.
- **Prefer pure functions > components > hooks** when extracting shared logic.
- **No business logic in `useEffect`** — effects sync with external systems. Move logic into the handler that sets the value.
- **Hoist stranglers to the top of the tree** — branch feature flags / experiments at a parent wrapper, never inside the existing component.

**TypeScript**
- **No `as` casts** — refactor so types flow naturally.
- **No `!` non-null assertions** — use `=== undefined` / `=== null` guards or optional chaining.
- **No `eslint-disable`, `@ts-ignore`, `@ts-expect-error`** — fix the root cause.
- **No default exports** — named exports only.

**Co-location**
- **Helpers travel with their only caller** — when extracting a sub-component, every helper the parent no longer calls moves with it. No orphans left behind.
- **Narrowest scope for variables** — declare in the block that uses them, not at the top of the function.
- **Extract cohesive UI clusters** — form fields, list rows, conditional sub-views with their own helpers (and optional state) belong in their own component. Cohesion is the trigger, not size; state is not required.
- **Derive at the point of use** — pass entities (or entity IDs) down; compute derived values where they're consumed. Don't pre-compute in the parent and thread results as props. Duplicating cheap derivation calls across consumers beats centralizing them upstream.

**Data safety**
- **No silent defaults** (`?? 0`, etc.) for money or any value whose absence is a programming bug. Throw or guard.

**GraphQL**
- **In render: `.useSuspended`**, never the plain hook. Ensure a `<Suspense>` boundary above.
- **In callbacks: `.fetchAsync`** inside the callback — don't hook at render time to use the result in a handler.
- **Conditional queries → conditional wrapper components**, never `skip`.
- **No explicit `fetchPolicy: 'cache-first'`** — it's the default; writing it is noise.

**Testing**
- **Never change implementation to make a test pass** — fix the test, not the code (except for genuine production bugs).
- **Keep tests in sync with code** — new fn → new test; modified → updated; moved → tests move too.

**i18n**
- **Namespace keys to location** — pattern: `flow::screen::component::detail`.
- **Update keys** when a component is renamed, split, or moved.
- **Never reuse keys across components** — duplicate the key even if the text is identical.

**Tooling**
- **`date-fns`, not `moment.js`**.
- **Never `git add` / `commit` / `push`** unless explicitly asked.

**General**
- **No comments** unless the *why* is non-obvious (hidden constraint, subtle invariant, external-bug workaround). Never narrate what code does or reference the current task/ticket/caller.
- **Refactor copied code** before using it — treat copy-paste as a first draft; apply all rules.
<!-- END PREFLIGHT -->

---

## Code structure

**No unnecessary indirection** — don't create variables, functions, or aliases that add a layer without adding meaning. Inline directly at the call site. Prefer repetition over a spurious abstraction.
- Don't: `const handleChange = (v) => ctx.setName(v); <Input onChange={handleChange} />`
- Do: `<Input onChange={ctx.setName} />`
- Don't: `const isOwner = user.role === 'owner'; if (isOwner) { ... }`
- Do: `if (user.role === 'owner') { ... }`
- Don't: `const label = item.name.trim(); return <Text>{label}</Text>`
- Do: `return <Text>{item.name.trim()}</Text>`

**Inline prop/param types** — extract a type alias only if the same shape appears in 2+ places. Top-level/exported components are not exempt — if `Props` is used exactly once (at the signature), inline it.
- Don't: `type FooProps = { name: string }; const Foo = (props: FooProps) => ...`
- Do: `const Foo = (props: { name: string }) => ...`
- Don't (still wrong, even when exported): `type Props = { cartId: string }; export const Checkout = (props: Props) => ...`
- Do: `export const Checkout = (props: { cartId: string }) => ...`

**Extract nested ternaries** — a single inline ternary is fine; two or more levels become a named pure function above the component.

**Positive conditions first** — when there's an `if/else` or two-branch ternary, put the positive/truthy case first. Applies to `=== null` / `=== undefined` checks in render ternaries — those are not guards, just inverted conditions. Only early-return guard clauses (`if (!x) { return; }`) are exempt.
- Don't: `!isReady ? fallback : content`
- Do: `isReady ? content : fallback`
- Don't: `value === null ? <Empty /> : <Display value={value} />`
- Do: `value !== null ? <Display value={value} /> : <Empty />`

**Named-param objects** — any regular function (not a component or hook) with 2+ params must use a single `params` object.
- Don't: `fn(a: A, b: B, c: C)`
- Do: `fn(params: { a: A; b: B; c: C })`

**Early returns over nested conditions** — validate preconditions at the top and return immediately; keep the happy path at the lowest indentation level. Deep nesting forces the reader to track multiple simultaneous conditions; each guard clause reduces that load for everything that follows.
- Don't: `if (user) { if (user.active) { if (hasPermission) { doWork(); } } }`
- Do: `if (!user) { return; } if (!user.active) { return; } if (!hasPermission) { return; } doWork();`

**No switch statements** — use `if` with early returns. Switches encourage fall-through bugs, require explicit `break`s, and obscure control flow. Early returns make each branch self-contained and the happy path obvious at the bottom.
- Don't: `switch (status) { case 'a': ...; break; case 'b': ...; break; default: ... }`
- Do: `if (status === 'a') { return ...; } if (status === 'b') { return ...; } return ...`

**No IIFEs in components** — extract `(() => { ... })()` as a named pure function above the component.

**Curly braces on every block** — wrap all `if`, `else`, `for`, `while` bodies in `{}`, even single-liners.

---

## React

**No `useCallback` / `useMemo` by default** — only add them for a specific, identifiable performance problem. Never preemptively.

**No destructuring hook returns or props** — always use the whole object. Namespaced access (`result.data`, `props.nav`) makes it clear at the read site where each value comes from, without needing to trace back to the destructure. Exception: tuple hooks (`useState`, `useReducer`, `useStorageState`) use array destructuring — that's their intended API.
- Don't: `const { data, loading } = useMyHook()` or `const MyComp = ({ nav, route }: Props) => ...`
- Do: `const result = useMyHook(); result.data;` or `const MyComp = (props: Props) => { props.nav; }`

**Prefer pure functions > components > hooks** when extracting shared logic. Reach for a hook only when React lifecycle is genuinely required.

**No business logic in `useEffect`** — effects sync with external systems. If an effect runs only because a value changed that you set elsewhere, move the logic into the handler that sets it.

**Hoist stranglers to the top of the tree** — when feature flagging, A/B testing, or toggling between an experimental and existing variant, place the branching logic as high as possible in the component tree. Mount the new variant from a parent wrapper; don't weave the conditional into the existing component's internals. The original code path stays unmodified, the experiment is trivially removable, the diff stays small.
- Don't: add `isExperiment` checks scattered through `<PaymentForm>` internals
- Do: in the parent, render `isExperiment ? <PaymentFormV2 /> : <PaymentForm />`
- Don't: thread a `variant` prop down through 3 layers so a leaf can branch
- Do: wrap at the highest common ancestor; each variant renders its own subtree

---

## TypeScript

**No `as` type casting** — refactor so types flow naturally.

**No non-null assertions `!`** — use `=== undefined` / `=== null` guards with early returns, or optional chaining.

**No `eslint-disable` or `@ts-ignore` / `@ts-expect-error`** — fix the underlying issue.

**No default exports** — named exports only.

---

## Co-location

Code that changes together should live together. Distance between related pieces forces the reader to scroll, search, or mentally hold context that should be local — every extra hop is a chance to lose the thread.

**Helpers travel with their only caller** — when a pure function, constant, or type alias is used by exactly one component (or sub-cluster), it belongs in that component's scope or file. Helpers stranded at the top of a parent file but called only from one child are noise the reader scans past. **When you extract a sub-component, every helper the parent no longer calls moves with it — never leave orphan helpers behind.** If multiple children use a helper, hoist it to the nearest shared scope, and no further. A utility used only within one file does not graduate to a shared module.

**Declare variables in the narrowest scope that uses them** — push every declaration down to the block where it's first read. A value used only inside one `if` branch belongs inside that branch, not at the top of the function. Top-of-function declarations are reserved for values that genuinely live across the whole body. Narrow scope makes the variable's lifetime obvious and stops the reader from tracking values that aren't yet relevant.
- Don't: declare `const formatted = format(date, 'yyyy-MM-dd');` at the top, then use it only inside one branch 30 lines later
- Do: declare `formatted` inside the branch that uses it
- Don't: top-of-function `let result;` reassigned inside each branch
- Do: `return` directly from each branch

**Extract cohesive UI clusters into their own component** — when a set of related JSX, the helpers it calls, and (if present) the state and handlers it touches are used only together and nowhere else in the parent, pull them into a child component. Cohesion is the trigger, not size, and **state is not required**.

Three patterns that almost always want extracting:
- **Form fields / inputs**: an input + its state + its `onChange` + surrounding markup that no other part of the parent reads
- **List rows**: the `<li>`/`<tr>`/row-shaped JSX inside a `.map(...)` that calls helpers used only by that row, even when the row has no state of its own
- **Conditional sub-views**: a branch of JSX rendered under one condition, with its own helpers and possibly its own state

The test: if every piece of the cluster — state, handlers, helpers, markup — can move into a child without the parent needing any of them, it belongs in the child. Don't lift state up "just in case" a sibling needs it later; wait until something actually does.

**Derive at the point of use** — when a child needs values derived from an entity (a formatted name, a computed status, a flag inferred from a few fields), pass the entity itself (or its ID) and derive at the consumer. Don't pre-compute derived values in the parent — or worse, at the API boundary — and thread the results down as props. Duplicating a one-line derivation across two consumers is preferable to centralizing it upstream.

Why: pre-computing pulls derivation logic away from where it's read, bloats prop interfaces, and couples the parent to every child's display needs. When the derivation rule changes (a new field, an edge case, a different format), the change belongs next to the JSX that renders it, not three layers up. Pure derivations are cheap — calling `formatName(user)` in three siblings costs nothing measurable and pays back every time someone reads the code.

- Don't: parent computes `const fullName = formatName(user); const isPremium = user.tier === 'gold';` and passes `fullName` and `isPremium` as props to children
- Do: parent passes `user` (or `userId`); each child calls `formatName(props.user)` / checks `props.user.tier === 'gold'` where the value is rendered
- Don't: an API-shaped object gets unpacked into a wide bag of pre-derived flags at the top of the screen, then passed down
- Do: pass the entity; let each leaf reach for the field or derivation it actually needs

Exception: if a derivation is measured-expensive, or the derived value *is* the contract of a reusable component (e.g., a generic `<Badge label={...} />`), passing the derived value is correct.

---

## Data safety

**Never default money amounts** — never use `?? 0` or any numeric fallback for a monetary value. A missing amount is a bug.
- Don't: `new Decimal(ctx.amount ?? 0).toFixed(2)`
- Do: `if (ctx.amount === undefined) { throw new Error('amount missing'); }` then use `ctx.amount`

Applies broadly: never silently default any value whose absence indicates a programming error.

---

## GraphQL

**In render: always `.useSuspended`** — never the plain hook. Ensure a `<Suspense>` boundary exists above.

**In callbacks: `.fetchAsync`** — if data is only needed inside a callback, call `.fetchAsync` inside it. Don't hook at render time to use the result in a handler.

**Conditional queries: conditional wrapper components, never `skip`** — write a wrapper component that only mounts when the condition is true.
- Don't: `MyQuery.useSuspended({ variables, skip: !condition })`
- Do: mount a `<DataExtractor />` component conditionally; query runs only when it mounts.

**No explicit `fetchPolicy: 'cache-first'`** — it's the default; writing it is noise.

---

## Testing

**Never change implementation to make a test pass** — fix the test, not the code. Only touch implementation for genuine production bugs.

**Keep tests in sync with code changes**:
- New functionality → write new tests
- Modified functionality → update existing tests
- Moved functionality → move the tests too; never leave them in the old location

---

## i18n

**Namespace keys to their location** — when moving code, update all i18n keys to the destination's namespace. Pattern: `flow::screen::component::detail`

**Keep keys in sync with component identity** — update i18n keys whenever a component is renamed, split into smaller components, or moved to a different scope. Keys must reflect the current component name and location, not the original one.

**Never reuse keys across components** — duplicate the key with a component-specific name, even if the text is identical. Shared keys are silent coupling.

---

## Tooling

**Use `date-fns` not `moment.js`**
- Don't: `import moment from 'moment'; moment(date).format('YYYY-MM-DD')`
- Do: `import { format } from 'date-fns'; format(date, 'yyyy-MM-dd')`

**Never commit, stage, or push on the user's behalf** — do not run `git add`, `git commit`, `git push`, or any equivalent unless explicitly asked.

---

## General

**Always refactor copied code** — treat copy-pasted code as a first draft; apply all rules before using it.

**No comments by default** — only add a comment when the *why* is non-obvious: a hidden constraint, a subtle invariant, or a workaround for a specific external bug. Never describe what the code does; never reference the current task, ticket, or caller.
