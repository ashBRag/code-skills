# Copilot Workspace Instructions

Applies to: test generation, JSDoc authoring, and Zod DTO authoring.

---

## JSDoc

Write JSDoc for every exported function, hook, and component.

Required tags in this order:

1. Description — one sentence, active voice, no "This function" or "This component".
2. `@param` — one per parameter, include type and what it represents.
3. `@returns` — what the return value represents, not just its type.
4. `@throws` — one per distinct error condition. State the type and when it is thrown. Required if the function throws under any condition.
5. `@example` — one minimal, runnable example. Use realistic values, not `foo`/`bar`.

Rules:

- No redundant type repetition if TypeScript types are already expressive.
- For hooks: document the returned object's fields individually if they are not self-evident from the type.
- For components: document non-obvious props and any side effects (e.g. "Triggers a query refetch on mount").
- Do not document internal or non-exported functions unless explicitly asked.

```ts
/**
 * Fetches a paginated list of orders for the given user.
 *
 * @param userId - The ID of the user whose orders are being fetched.
 * @param options - Pagination options: `page` (1-indexed) and `limit`.
 * @returns A TanStack Query result containing a page of orders with total count and cursor.
 * @throws {AppError} If the API response fails schema validation.
 * @example
 * const { data, isPending } = useUserOrders("usr_123", { page: 1, limit: 20 });
 */
export function useUserOrders(
  userId: string,
  options: PaginationOptions
): UseSuspenseQueryResult<PagedResult<Order>> {}
```

---

## Zod DTOs

- One schema per file unless schemas are tightly coupled (e.g. request + response pair from the same endpoint).
- Export the schema and its inferred type from the same file.
- Name the schema `<Entity>Schema` and the type `<Entity>`.
- Use `.describe()` on every field — feeds OpenAPI tooling, error messages, and serves as documentation.
- Prefer `.min()`, `.max()`, `.regex()`, `.email()` over custom `.refine()` where Zod builtins suffice.
- Use `.refine()` only for cross-field validation or domain rules not expressible otherwise. Always provide a `message` and `path`.
- Never use `z.any()`. Use `z.unknown()` and narrow explicitly downstream.
- Coerce only at the trust boundary (URL params, query strings, `localStorage` reads). Never coerce inside business logic schemas.
- For API response schemas: validate with `.safeParse()` at the generated client boundary. Do not use `.parse()` and let it throw unhandled.

```ts
import { z } from "zod";

export const CreateOrderSchema = z.object({
  userId: z.string().uuid().describe("ID of the user placing the order"),
  items: z
    .array(
      z.object({
        productId: z.string().uuid().describe("ID of the product"),
        quantity: z.number().int().min(1).describe("Number of units ordered"),
      })
    )
    .min(1)
    .describe("Line items in the order"),
  couponCode: z
    .string()
    .regex(/^[A-Z0-9]{6,12}$/)
    .optional()
    .describe("Optional promotional coupon code"),
});

export type CreateOrder = z.infer<typeof CreateOrderSchema>;
```

---

## Tests

Stack: **Vitest + React Testing Library + msw**. Co-locate tests: `MyModule.test.ts` beside `MyModule.ts`.

### Structure

- One `describe` block per module, hook, or component.
- Group by behaviour, not by function name.
- Test names: `"does X when Y"` — observable outcome first, condition second.
- Each test gets a fresh `QueryClient` — never share instances across tests.

### What to mock

- Mock at the network boundary only: HTTP via `msw` handlers. Do not mock `fetch` directly.
- Do not mock: internal utility functions, Zod schemas, `AppError`, query key factories.
- Do not mock `useQuery`, `useMutation`, `useSuspenseQuery`, or any TanStack Query hook — test through them with real `QueryClient` instances.
- For hooks that call the API: wrap in `renderHook` with a `QueryClientProvider`. Let msw intercept the network call.
- For optimistic update paths: assert on intermediate UI state before the msw response resolves, then on the settled state after.

### Setup

```ts
// test-utils/renderWithClient.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { render, renderHook } from "@testing-library/react";

function makeTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
}

export function renderWithClient(ui: React.ReactElement) {
  const client = makeTestQueryClient();
  return render(
    <QueryClientProvider client={client}>{ui}</QueryClientProvider>
  );
}

export function renderHookWithClient<T>(hook: () => T) {
  const client = makeTestQueryClient();
  return renderHook(hook, {
    wrapper: ({ children }) => (
      <QueryClientProvider client={client}>{children}</QueryClientProvider>
    ),
  });
}
```

### What to assert

- Assert on rendered output and user-observable state — not internal calls, query cache state, or implementation details.
- For mutation hooks: assert on UI changes after `mutate` resolves — not on whether `invalidateQueries` was called.
- Error paths: assert the error UI renders correctly; assert `Logger` was called with the original error if applicable.
- Optimistic updates: assert the optimistic value appears before network settlement, and the rolled-back value appears on error.
- Do not assert on the number of times a mock was called unless call count is the behaviour under test.
- For `useActionState` / `useFormStatus` flows: assert on the rendered form state (disabled, pending, field errors shown) — not on action function invocation.

### Example — query hook

```ts
import { server } from "@/test-utils/msw-server";
import { http, HttpResponse } from "msw";
import { renderHookWithClient } from "@/test-utils/renderWithClient";
import { useUserOrders } from "./useUserOrders";

describe("useUserOrders", () => {
  it("returns orders when the API responds successfully", async () => {
    server.use(
      http.get("/api/users/:userId/orders", () =>
        HttpResponse.json({ items: [{ id: "ord_1" }], total: 1 })
      )
    );

    const { result } = renderHookWithClient(() =>
      useUserOrders("usr_123", { page: 1, limit: 20 })
    );

    await waitFor(() => expect(result.current.data).toBeDefined());
    expect(result.current.data?.items).toHaveLength(1);
  });

  it("surfaces an error state when the API returns 404", async () => {
    server.use(
      http.get("/api/users/:userId/orders", () =>
        HttpResponse.json({ message: "Not found" }, { status: 404 })
      )
    );

    const { result } = renderHookWithClient(() =>
      useUserOrders("usr_999", { page: 1, limit: 20 })
    );

    await waitFor(() => expect(result.current.isError).toBe(true));
  });
});
```

### Example — mutation with optimistic update

```ts
describe("useCancelOrder", () => {
  it("shows cancelled state optimistically before server confirms", async () => {
    let resolve!: () => void;
    server.use(
      http.post(
        "/api/orders/:orderId/cancel",
        () =>
          new Promise((res) => {
            resolve = () => res(HttpResponse.json({ status: "cancelled" }));
          })
      )
    );

    renderWithClient(<OrderCard orderId="ord_1" />);
    await userEvent.click(screen.getByRole("button", { name: /cancel/i }));

    expect(screen.getByText("Cancelling…")).toBeInTheDocument();
    resolve();
    await waitFor(() =>
      expect(screen.getByText("Cancelled")).toBeInTheDocument()
    );
  });

  it("rolls back optimistic state on network failure", async () => {
    server.use(
      http.post("/api/orders/:orderId/cancel", () =>
        HttpResponse.json({ message: "Server error" }, { status: 500 })
      )
    );

    renderWithClient(<OrderCard orderId="ord_1" />);
    await userEvent.click(screen.getByRole("button", { name: /cancel/i }));

    await waitFor(() =>
      expect(
        screen.getByRole("button", { name: /cancel/i })
      ).toBeInTheDocument()
    );
  });
});
```

### Example — form with useActionState

```ts
describe("CreateOrderForm", () => {
  it("displays field errors when items array is empty", async () => {
    renderWithClient(<CreateOrderForm />);
    await userEvent.click(screen.getByRole("button", { name: /submit/i }));

    await waitFor(() =>
      expect(
        screen.getByText(/at least one item required/i)
      ).toBeInTheDocument()
    );
  });

  it("disables submit button while submission is pending", async () => {
    server.use(
      http.post("/api/orders", () => new Promise(() => {})) // never resolves
    );

    renderWithClient(<CreateOrderForm />);
    await userEvent.click(screen.getByRole("button", { name: /submit/i }));

    expect(screen.getByRole("button", { name: /submit/i })).toBeDisabled();
  });
});
```
