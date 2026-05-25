# Project Rules

These rules apply to all AI agents operating in this codebase.

## STOP Gates

Hard interrupts. Do not proceed past any of these without explicit human confirmation (`"yes"`, `"proceed"`, `"approved"`, or equivalent) in the current session. Flag it and STOP.

- Change touches more than one file → list all affected files, then STOP
- A new dependency is needed
- An existing interface or contract would change
- The correct approach is genuinely ambiguous
- A lint, type, or test failure is encountered mid-task
- A structural file has no readable contract
- Change touches auth, token handling, cryptography, or role enforcement
- **Library resolution:** check `package.json` before referencing any library. If absent, flag and STOP.
- **Test gate:** if a test file exists, state "Test file found. Update to cover the change, leave as-is, or update tests only?" and STOP. If none exists, state "No test file found. Proceed without tests, or write tests first?" and STOP. Never silently skip or add tests.

## Context Gathering

Before writing any code, read in this order and stop at the first source that gives sufficient context:

1. OpenAPI spec or generated client types for any endpoint being touched.
2. Co-located test file (`*.test.ts` / `*.test.tsx`).
3. JSDoc on the target module and its direct dependencies.
4. DTO or schema definitions (Zod schemas, generated types).
5. Code implementation (last resort).

## Intent Before Code

Before any implementation, state in three bullets or fewer:

- What the unit does.
- What constraints or contracts apply (from types, tests, or OpenAPI spec).
- What the proposed change affects.

STOP. Wait for explicit confirmation before writing any code.  
Skip only for pure stateless transformations with no dependencies.

## Output Constraints

- Present changes as a diff with ±5 lines of context. Do not auto-apply.
- Never scaffold or create new files unless explicitly instructed.

## Security

- Validate all external inputs at the trust boundary using Zod: form submissions, URL params, route state, query strings, localStorage reads.
- Never trust client-supplied user IDs, roles, or permissions — derive from decoded token claims, and re-validate on every protected operation.
- Enforce ownership checks before any data read or mutation, even if the API enforces them — document the assumption explicitly if deferred.
- No string interpolation in dynamic URLs, query params, or template-driven APIs — use typed URL builders or parameterized constructors.
- No secrets, API keys, or auth tokens in source code, comments, or committed `.env` files. Only `VITE_` prefixed vars are safe to expose to the browser — treat this as a security rule.
- Never surface internal error messages, stack traces, or system paths to UI.
- Strip or mask PII before passing to any external logging or analytics sink.
- Tokens must be stored in memory (not `localStorage`) unless there is an explicit, documented tradeoff accepted by the team.

## TypeScript

- Prefer `interface` for object shapes; `type` for unions, intersections, and mapped types.
- Never use non-null assertion (`!`) unless a preceding guard makes it provably safe.
- Derive client-side types from the OpenAPI spec (generated via `openapi-typescript` or equivalent) — do not hand-write types that duplicate the spec.
- Export types from the module that owns them; re-export from `index.ts` if needed.
- If interface merging or module augmentation is intentional, document it at the declaration site.

## React 19

> Pinned: **React 19.x**

### Component Model

- Function components only. No class components.
- Default to no `"use client"` annotation (SPA context — all components are client components, but mark intent where relevant in comments for future SSR compatibility).
- No prop drilling beyond two levels — use composition, context, or co-located state.

### React 19 Primitives

- **`use(promise)`** — permitted for reading promises or context inside render. Do not use inside conditionals.
- **`useOptimistic`** — preferred over manual optimistic state for mutations that have a clear rollback shape.
- **`useFormStatus`** — use inside form subtrees to reflect pending state; replaces manual `isSubmitting` flags in form components.
- **`useActionState`** — use for form-bound actions that return structured results (validation errors, success state). Do not manage form result state with `useState + handler` when this applies.
- **`useTransition`** — wrap non-urgent state updates (navigation, tab switches, search) to keep input responsive. Do not use as a general debounce substitute.
- **`useDeferredValue`** — use to defer expensive derived renders (filtered lists, heavy charts). Requires the deferred consumer to be wrapped in `memo`.
- **Ref as prop** — React 19 forwards refs automatically. Do not use `forwardRef` for new components; remove it when refactoring existing ones.
- **`use client` directives** — not applicable in a pure SPA, but add `// @ssr-unsafe` comments on any component using `window`, `document`, or `navigator` directly.

### Suspense

- Wrap async boundaries explicitly — do not rely on implicit suspension.
- Default: one `<Suspense>` per route-level component.
- Move lower only when a subset of the UI is meaningfully useful without the deferred data — document why with a comment.
- Do not wrap every async component individually.
- Pair every `<Suspense>` with an `<ErrorBoundary>`. Never leave a Suspense boundary without an error fallback.

### Transitions & Concurrent Features

- Treat any state update that triggers a fetch or heavy recompute as a candidate for `useTransition`.
- Never use `startTransition` to silence Suspense waterfalls — fix the data dependency instead.

## Data Fetching — TanStack Query v5

- TanStack Query owns **all server state**: fetching, caching, refresh, invalidation, optimistic updates.
- Never replicate server state with `useState + useEffect` — replace with `useQuery` if that pattern appears.
- Query keys must be structured arrays, defined in a co-located `queryKeys.ts` (or equivalent key factory). No inline string keys.
- Always set `staleTime` explicitly. Never rely on the default (`0`) in production code — document the chosen value and why.
- Use `useSuspenseQuery` in components wrapped by a `<Suspense>` boundary. Use `useQuery` only when you need to handle loading/error state inline (justify in a comment).
- Mutations via `useMutation`. On `onSuccess`, invalidate or update the query cache explicitly — never refetch by side-effect timing.
- Optimistic updates via `useMutation`'s `onMutate` / `onError` rollback pattern, or `useOptimistic` for form-bound flows.
- Prefetch in route loaders or parent components when data is known to be needed — do not waterfall sibling queries.
- Never pass query data as props to seed sibling or child query state. Each consumer calls `useQuery` with the same key; the cache deduplicates.

## OpenAPI Contract

- All API interactions must go through a generated client or typed fetch wrapper derived from the OpenAPI spec. No ad-hoc `fetch` calls with hand-written URLs.
- Regenerate the client when the spec changes — treat the spec as the source of truth, not the implementation.
- If the spec and implementation diverge, flag it and STOP. Do not paper over it with a cast.
- Validate response shapes with Zod at the boundary if the generated client does not already do so — particularly for partial or `anyOf` schemas.
- Never cast an API response with `as SomeType` without a preceding runtime check or generated type guard.

## State Management

Resolve in this order:

1. **Server state?** → TanStack Query (`useQuery` / `useMutation`).
2. **Local to ≤ 2 levels?** → `useState` / `useReducer`.
3. **Form state?** → `useActionState` + `useFormStatus` (React 19 native) or a form library (e.g. React Hook Form) for complex schemas. Not both.
4. **Stable, cross-cutting config?** → `useContext` with split contexts by update frequency. Memoize value with `useMemo` if the provider re-renders often.
5. **Complex, global, client-only, shared across disconnected subtrees?** → Redux (RTK only). Justified only when Query + Context cannot meet the need.

## Forms

- Use `useActionState` for forms with server-round-trip semantics (submission → validation → result).
- Use `useFormStatus` inside form subtrees to reflect pending state.
- Validate inputs with Zod before submission — errors returned as structured `fieldErrors`, never thrown to an error boundary.
- Do not mix controlled and uncontrolled inputs in the same form.
- Never use `e.preventDefault()` + manual fetch in place of an action unless explicitly justified.

## Environment Variables

- Validate all env vars at startup with Zod in a single `env.ts`. Import from there; never access `import.meta.env` directly elsewhere.
- `VITE_` prefix only for values explicitly safe to expose to the browser. Treat as a security rule.
- Document each variable's purpose and sensitivity level in a committed `.env.example`.

## Error Handling

- Catch errors at the boundary closest to the failure.
- Log the full error (including stack) with the project logger before any re-throw or user-facing render.
- Re-throw unexpected errors as `AppError`: `{ message: string; data?: unknown }`.
  - `message`: human-readable, caller-safe — no stack or internals.
  - `data`: structured context (IDs, inputs) relevant to the failure.
- Use `instanceof AppError` for upstream narrowing (error boundaries, query `onError`).
- TanStack Query errors surface to the nearest `<ErrorBoundary>` when using `useSuspenseQuery` — ensure every Suspense boundary has one.
- Validation errors from form actions are not `AppError` — return as structured `fieldErrors`.

## Testing

- Unit: pure functions, Zod schemas, query key factories, utility hooks.
- Integration: component + query interaction via `renderWithClient` (custom wrapper with a test `QueryClient`).
- Avoid mocking `fetch` globally — use `msw` (Mock Service Worker) to intercept at the network boundary.
- Every `useMutation` must have a test covering the `onError` rollback path if it uses optimistic updates.
