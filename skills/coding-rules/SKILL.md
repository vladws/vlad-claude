---
name: coding-rules
description: Opinionated mandatory coding rules for TypeScript/React development. Contains non-negotiable standards covering code structure, React patterns, TypeScript safety, GraphQL, testing, i18n, and tooling. Consult this skill whenever writing or modifying any TypeScript or React code — components, hooks, GraphQL queries, tests, i18n strings — even if the user hasn't explicitly asked about coding standards. Also use when reviewing code, refactoring, or when something "looks off." These rules override default coding instincts.
---

# Coding Rules

Apply to all code written or modified. Verify every changed file complies before finishing any task.

---

## Code Structure

**Curly braces on every block** — wrap all `if`, `else`, `for`, `while` bodies in `{}`, even single-liners.

**Positive conditions first** — when there's an `if/else` or ternary with two branches, put the positive/truthy case first. Guard clauses (`if (!x) { return; }`) are exempt.
- Don't: `!isReady ? fallback : content`
- Do: `isReady ? content : fallback`

**No IIFEs in components** — extract `(() => { ... })()` as a named pure function above the component.

**Extract nested ternaries** — a single inline ternary is fine; two or more levels must become a named pure function above the component.

**No unnecessary indirection** — don't create variables, functions, or aliases that add a layer without adding meaning. Inline directly at the call site. Prefer repetition over a spurious abstraction.
- Don't: `const handleChange = (v) => ctx.setName(v); <Input onChange={handleChange} />`
- Do: `<Input onChange={ctx.setName} />`
- Don't: `const save = () => storage.save(); save();`
- Do: `storage.save();`
- Don't: `function formatDate(d: Date) { return toISOString(d); }`
- Do: call `toISOString(d)` directly
- Don't: `const isOwner = user.role === 'owner'; if (isOwner) { ... }`
- Do: `if (user.role === 'owner') { ... }`
- Don't: `const label = item.name.trim(); return <Text>{label}</Text>`
- Do: `return <Text>{item.name.trim()}</Text>`

**Named-param objects** — any regular function (not a component or hook) with 2+ params must use a single `params` object.
- Don't: `fn(a: A, b: B, c: C)`
- Do: `fn(params: { a: A; b: B; c: C })`

**Inline prop/param types** — extract a type alias only if the same shape appears in 2+ places.
- Don't: `type FooProps = { name: string }; const Foo = (props: FooProps) => ...`
- Do: `const Foo = (props: { name: string }) => ...`

---

## React

**No destructuring hook returns or props** — always use the whole object. Namespaced access (`result.data`, `props.nav`) makes it clear at the read site where each value comes from, without needing to trace back to the destructure. Exception: tuple hooks (`useState`, `useReducer`, `useStorageState`) use array destructuring — that's their intended API.
- Don't: `const { data, loading } = useMyHook()` or `const MyComp = ({ nav, route }: Props) => ...`
- Do: `const result = useMyHook(); result.data;` or `const MyComp = (props: Props) => { props.nav; }`

**No business logic in `useEffect`** — effects are for syncing with external systems. If an effect runs only because a value changed that you set elsewhere, move the logic into the handler that sets it.

**No `useCallback`/`useMemo` by default** — only add them for a specific, identifiable performance problem. Never preemptively.

**Prefer pure functions > components > hooks** when extracting shared logic. Reach for a hook only when React lifecycle is genuinely required.

---

## TypeScript

**No `as` type casting** — refactor so types flow naturally.

**No non-null assertions `!`** — use `=== undefined`/`=== null` guards with early returns or optional chaining instead.

**No `eslint-disable` or `@ts-ignore`/`@ts-expect-error`** — fix the underlying issue.

**No default exports** — named exports only.

---

## Data Safety

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
