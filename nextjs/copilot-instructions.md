# Copilot Workspace Instructions

Applies to: test generation, JSDoc authoring, and Zod DTO authoring.

---

## JSDoc

Write JSDoc for every exported function and method.

Required tags in this order:

1. Description — one sentence, active voice, no "This function".
2. `@param` — one per parameter, include type and what it represents.
3. `@returns` — what the return value represents, not just its type.
4. `@throws` — one per distinct error condition. State the type and when it is thrown.
5. `@example` — one minimal, runnable example. Use realistic values, not `foo`/`bar`.

Rules:

- No redundant type repetition if TypeScript types are already expressive.
- `@throws` is required if the function throws under any condition — do not omit.
- `@example` must reflect the actual call signature including required options.
- Do not document internal or non-exported functions unless explicitly asked.

```ts
/**
 * Fetches a paginated list of orders for the given user.
 *
 * @param userId - The ID of the user whose orders are being fetched.
 * @param options - Pagination options: `page` (1-indexed) and `limit`.
 * @returns A page of orders with total count and cursor.
 * @throws {NotFoundError} If no user exists with the given ID.
 * @throws {UnauthorizedError} If the caller lacks read access to the user's orders.
 * @example
 * const result = await fetchUserOrders("usr_123", { page: 1, limit: 20 });
 */
export async function fetchUserOrders(
  userId: string,
  options: PaginationOptions
): Promise<PagedResult<Order>> {}
```

---

## Zod DTOs

- One schema per file unless schemas are tightly coupled (e.g. request + response pair).
- Export the schema and its inferred type from the same file.
- Name the schema `<Entity>Schema` and the type `<Entity>`.
- Use `.describe()` on every field — this doubles as documentation and feeds error messages.
- Prefer `.min()`, `.max()`, `.regex()`, `.email()` over custom `.refine()` where Zod builtins suffice.
- Use `.refine()` only for cross-field validation or domain rules not expressible otherwise. Always provide a `message` and `path`.
- Never use `z.any()`. Use `z.unknown()` and narrow explicitly downstream.
- Coerce only at the trust boundary (e.g. query params). Never coerce inside business logic schemas.

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

Stack: Vitest + React Testing Library. Co-locate tests: `MyModule.test.ts` beside `MyModule.ts`.

### Structure

- One `describe` block per module or component.
- Group by behaviour, not by function name.
- Test names: `"does X when Y"` — observable outcome first, condition second.

### What to mock

- Mock at the network/IO boundary only: `fetch` (via `msw` or `vi.fn()`), database clients, external SDK calls.
- Do not mock: internal utility functions, Zod schemas, `AppError`.
- Server Actions: import and call directly. Mock only their external dependencies, not the action itself.
- TanStack Query: wrap component under test in `QueryClientProvider` with a fresh `QueryClient` per test. Do not mock `useQuery` or `useMutation`.

### What to assert

- Assert on rendered output and user-observable state — not internal calls or implementation details.
- Server Actions: assert on the returned `ActionResult` shape (`ok`, `fieldErrors`, `message`).
- Error paths: assert `ok: false` with the correct `fieldErrors` or `message`; assert `Logger` was called with the original error.
- Do not assert on the number of times a mock was called unless call count is the behaviour under test.

### Example — Server Action

```ts
describe("createOrder", () => {
  it("returns fieldErrors when items array is empty", async () => {
    const result = await createOrder({ userId: "usr_123", items: [] });
    expect(result.ok).toBe(false);
    expect(result.fieldErrors?.items).toBeDefined();
  });

  it("throws and logs on unexpected DB failure", async () => {
    mockDb.insert.mockRejectedValueOnce(new Error("connection lost"));
    await expect(createOrder(validPayload)).rejects.toBeInstanceOf(AppError);
    expect(Logger.error).toHaveBeenCalledWith(expect.any(Error));
  });
});
```
