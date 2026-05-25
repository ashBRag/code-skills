# Folder Structure

## Tree

```
src/
├── app/                        # Router configuration and route components
│   ├── routes/                 # One file per route: named <RouteName>.route.tsx
│   ├── router.tsx              # createBrowserRouter definition
│   └── App.tsx                 # Root component; mounts RouterProvider
│
├── components/                 # Shared, stateless, domain-agnostic UI primitives
│   ├── button/
│   ├── card/
│   ├── container/
│   ├── form/
│   │   ├── radio-group/
│   │   ├── select/
│   │   ├── status-bar/
│   │   └── text-field/
│   ├── icon/
│   ├── menu/
│   ├── navList/
│   ├── scroll/
│   ├── tabs/
│   ├── timeline/
│   ├── toast/
│   └── wrapper/
│
├── sections/                   # Feature-scoped composition units (sub-page chunks)
│   └── <feature>/              # e.g. orders/, profile/, dashboard/
│       ├── components/         # Components private to this section
│       ├── hooks/              # useQuery / useMutation hooks for this feature
│       ├── schemas/            # Zod DTOs for this feature's API payloads
│       ├── types/              # Types private to this section
│       ├── utils/              # Pure functions private to this section
│       └── index.tsx           # Public surface — export only what routes consume
│
├── config/                     # Static app configuration (feature flags, constants)
├── hooks/                      # Shared hooks reused across 2+ sections
├── lib/                        # Third-party client setup (queryClient, apiClient, msw)
├── providers/                  # React context providers (auth, theme, locale)
├── styles/
│   ├── globals.css
│   └── profiles/
├── types/                      # Shared types reused across 2+ sections
├── utils/                      # Shared pure utilities reused across 2+ sections
└── env.ts                      # Zod-validated env vars — import from here only
```

## Placement Rules

**`components/`**
- Domain-agnostic only. No API calls, no TanStack Query hooks, no business logic.
- If a component needs a query or references a domain type, it belongs in `sections/<feature>/components/` instead.

**`sections/<feature>/`**
- All feature-specific code is co-located here: components, hooks, schemas, types, utils.
- Only `index.tsx` is the public surface. Routes import from the index; never from internal paths directly.
- A section must not import from another section. Shared code is promoted to a top-level directory.

**`hooks/`, `utils/`, `types/` (top-level)**
- Promoted from a section only when reused across 2 or more sections.
- If it exists only to serve one section, it stays co-located.

**`lib/`**
- `queryClient.ts` — single `QueryClient` instantiation; imported by `providers/` and test utils.
- `apiClient.ts` — generated OpenAPI client or typed fetch wrapper; the only place raw HTTP is constructed.
- `msw/` — handler setup and `worker.ts`; never imported in production paths.

**`providers/`**
- One file per context boundary (e.g. `AuthProvider.tsx`, `ThemeProvider.tsx`).
- Providers are composed in `App.tsx` in dependency order. Do not nest providers inside route components.

**`app/routes/`**
- One file per route: `Orders.route.tsx`, `Profile.route.tsx`.
- Route components are thin: import from `sections/`, wire loaders, render layout. No business logic.
- Loader functions (`loader`) defined in the route file. Data fetching via `queryClient.ensureQueryData` (prefetch pattern).

**`env.ts`**
- Single source for all `import.meta.env` access. Validated with Zod at module load.
- No other file accesses `import.meta.env` directly.

## Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Route component | `<Name>.route.tsx` | `Orders.route.tsx` |
| Section index | `index.tsx` | `sections/orders/index.tsx` |
| Query hook | `use<Entity><Action>.ts` | `useOrdersList.ts` |
| Mutation hook | `use<Action><Entity>.ts` | `useCancelOrder.ts` |
| Zod schema | `<Entity>Schema` / `<Entity>.schema.ts` | `CreateOrderSchema` |
| Context provider | `<Name>Provider.tsx` | `AuthProvider.tsx` |
| Shared util | camelCase | `formatCurrency.ts` |
| Test file | co-located, same name | `useOrdersList.test.ts` |