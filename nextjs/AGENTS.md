# Project Rules

These rules apply to all AI agents operating in this codebase.

## STOP Gates

Hard interrupts. Do not proceed past any of these without explicit human confirmation (`"yes"`, `"proceed"`, `"approved"`, or equivalent) in the current session. Flag it and the STOP

- Change touches more than one file → list all affected files, then STOP
- A new dependency is needed
- An existing interface or contract would change
- The correct approach is genuinely ambiguous
- A lint, type, or test failure is encountered mid-task
- A structural file has no readable contract
- Change touches auth, session handling, cryptography, or role enforcement
- **Library resolution:** check `package.json` before referencing any library. If absent, flag and STOP.
- **Test gate:** if a test file exists, state "Test file found. Update to cover the change, leave as-is, or update tests only?" and STOP. If none exists, state "No test file found. Proceed without tests, or write tests first?" and STOP. Never silently skip or add tests.

## Context Gathering

Before writing any code, read in this order and stop at the first source that gives sufficient context:

1. For any `@dev-library` package, read its `index.ts` JSDoc.
2. Co-located test file (`*.test.ts` / `*.test.tsx`).
3. JSDoc on the target module and its direct dependencies.
4. DTOs, Swagger spec, or named contracts.
5. Code implementation (last resort).

## Intent Before Code

Before any implementation, state in three bullets or fewer:

- What the unit does.
- What constraints or contracts apply (from types, tests, or spec).
- What the proposed change affects.

STOP. Wait for explicit confirmation before writing any code.  
Skip only for pure stateless transformations with no dependencies.

## Output Constraints

- Present changes as a diff with ±5 lines of context. Do not auto-apply.
- Never scaffold or create new files unless explicitly instructed.
- Before placing any file, read `architecture.md` Folder Structure.

## Security

- Validate all external inputs at the trust boundary with Zod: Server Actions, Route Handlers, query params, path segments, headers, cookies.
- Never trust client-supplied user IDs, roles, or permissions — derive from session/token server-side on every request.
- Enforce ownership checks before any data read or mutation.
- No string interpolation in SQL, shell commands, or template-driven APIs — use parameterized queries or ORM abstractions.
- No secrets in source code, comments, or committed `.env` files.
- Never return internal error messages, stack traces, or system paths to the client.
- Strip or mask PII from logs before they reach any external sink.

## TypeScript

- Prefer `interface` for object shapes; `type` for unions, intersections, and mapped types.
- Never use non-null assertion (`!`) unless a preceding guard makes it provably safe.
- If interface merging or module augmentation is intentional, document it at the declaration site.
- Export types from the module that owns them; re-export from `index.ts` if needed.

## React / Next.js

> Pinned: **Next.js 15.x**, App Router.

- Server Components are the default. Add `"use client"` only when state, effects, or browser APIs are required.
- No prop drilling beyond two levels — use composition or context.
- Async Server Components fetch data inline; never use `useEffect` for data fetching.
- `loading.tsx` and `error.tsx` required for every route segment that fetches data.
- `generateStaticParams` required for dynamic segments statically knowable at build time. Document the `dynamicParams` fallback decision as a comment in the route segment.

### Client/Server Boundary

Props crossing `"use client"` must be serializable: no functions, class instances, `Date`, `Map`, `Set`, or `undefined` in object positions. Use plain JSON-compatible types, or transfer as strings and parse on the client. If a value cannot be serialized, lift the logic into the Server Component or pass through a context provider inside the client subtree.

## Data Fetching & Caching

- Always state cache intent explicitly — never rely on defaults.
- Server Component fetching: use `fetch` with `cache` or `next.revalidate` inline.
- Within-render deduplication: use `React.cache()`.
- Invalidation after mutation: use `revalidatePath` or `revalidateTag` inside Server Actions or Route Handlers.

## Data Mutations

- **Server Actions** — mutations from the component tree. Co-locate in the route segment or a sibling `actions.ts`.
- **Route Handlers** — HTTP-addressable endpoints only: webhooks, third-party callbacks, non-browser clients.
- Never call a Route Handler from your own client components to perform a mutation.
- Validate all Server Action inputs with Zod at the top of the action, before any business logic.

### Server Action Return Shape

All Server Actions must return a discriminated union:

- Validation errors (`fieldErrors` populated): return via `useActionState`. Do not throw.
- Unexpected failures (`fieldErrors: null`): throw so the nearest error boundary catches. Log the full error with `Logger` before throwing.
- Never return raw `Error` instances or stack traces to the client.

## State Management

Resolve in this order:

1. **Server state?** → TanStack Query (`useQuery` / `useMutation`).
2. **Local to ≤ 2 levels?** → `useState` / `useReducer`.
3. **Stable, cross-cutting config?** → `useContext`.
4. **Complex, global, client-only, shared across disconnected subtrees?** → Redux (RTK only).

### TanStack Query

- Owns all server state: fetching, caching, refresh, invalidation, optimistic updates.
- Never replicate with `useState + useEffect` — replace with Query if that pattern appears.
- Client-side refetch/invalidation pattern: `prefetchQuery` in Server Component → `HydrationBoundary` + `dehydrate(queryClient)` → `useSuspenseQuery` in consumer.
- Never pass server-fetched data as props to seed client query state.

### useContext

- Use for stable, low-frequency values: auth identity, theme, locale, feature flags.
- Split contexts by update frequency. Memoize value with `useMemo` if the provider re-renders often.

### Redux

- Justified only when Query + Context cannot meet the need.
- Narrow scope in RSC apps: multi-step wizard state, undo/redo, real-time collaborative state.
- RTK only. No hand-rolled reducers or action creators.

## Suspense Boundaries

- Default: one boundary per route segment at the segment root.
- Move lower only when a subset of UI is meaningfully useful without deferred data — document why with a comment.
- Move higher only when two or more async components always resolve together.
- Do not wrap every async component individually.

## Environment Variables

- Validate all env vars at startup with Zod in a single `env.ts`. Import from there; never access `process.env` directly elsewhere.
- `NEXT_PUBLIC_` prefix only for values explicitly safe to expose to the browser. Treat as a security rule.
- For server-only vars, use the `server-only` package to hard-fail on accidental client imports.

## Error Handling

- Catch errors at the boundary closest to the failure.
- Log the full error (including stack) with `Logger` before any re-throw or return.
- Re-throw unexpected errors as `AppError`: `{ message: string; data?: unknown }`.
  - `message`: human-readable, caller-safe — no stack or internals.
  - `data`: structured context (IDs, inputs) relevant to the failure.
- Use `instanceof AppError` for upstream narrowing (error boundaries, middleware).
- Validation errors from Server Actions are not `AppError` — return as structured `fieldErrors`.

## Middleware

- Permitted: auth guards, redirects, locale detection, edge-safe logic.
- Always export a `matcher` config. Never run on all routes by default.
- Auth re-validate identity inside Server Actions and Route Handlers — middleware guards alone are not sufficient.
