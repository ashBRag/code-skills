# Project Rules

These rules apply to all AI agents operating in this codebase.

## STOP Gates

Hard interrupts. Do not proceed past any of these without explicit human confirmation (`"yes"`, `"proceed"`, `"approved"`, or equivalent) in the current session. Flag it and STOP.

- Change touches more than one file → list all affected files, then STOP
- A new dependency is needed
- An existing interface, DTO, or contract would change
- The correct approach is genuinely ambiguous
- A lint, type, or test failure is encountered mid-task
- A structural file has no readable contract
- Change touches auth, guards, interceptors, cryptography, or role enforcement
- Change touches Kafka producers/consumers, Redis, or AWS SDK usage
- **Library resolution:** check `package.json` first. If absent, check `@dev-libraries` index. If absent in both, flag and STOP.
- **`@dev-libraries` resolution order:** for auth, rate limiting, DB connection, metrics, logger, AWS SDK, Kafka, or Redis — always use the `@dev-libraries` package. Never introduce an external alternative without explicit approval.
- **Test gate:** if a test file exists, state "Test file found. Update to cover the change, leave as-is, or update tests only?" and STOP. If none exists, state "No test file found. Proceed without tests, or write tests first?" and STOP. Never silently skip or add tests.

---

## Context Gathering

Before writing any code, read in this order and stop at the first source that gives sufficient context:

1. For any `@dev-libraries` package, read its `index.ts` JSDoc first — this is authoritative.
2. Co-located test file (`*.spec.ts`).
3. JSDoc on the target module and its direct dependencies.
4. DTOs, Swagger decorators, or named contracts.
5. Code implementation (last resort).

---

## Intent Before Code

Before any implementation, state in three bullets or fewer:

- What the unit does.
- What constraints or contracts apply (from types, tests, or spec).
- What the proposed change affects.

STOP. Wait for explicit confirmation before writing any code.  
Skip only for pure stateless transformations with no dependencies.

---

## Output Constraints

- Present changes as a diff with ±5 lines of context. Do not auto-apply.
- Never scaffold or create new files unless explicitly instructed.
- Before placing any file, read `/docs/architecture.md` Folder Structure.

---

## Security

- Validate all external inputs at the trust boundary with `class-validator` + `ValidationPipe` (global, `whitelist: true`, `forbidNonWhitelisted: true`). Apply to all Controllers: body, query params, path params, and headers.
- Never trust client-supplied user IDs, roles, or permissions — derive from the authenticated request identity (JWT payload or session) server-side on every request. Use `@dev-libraries/auth` exclusively for token verification and identity extraction.
- Enforce ownership and authorization checks inside the Service layer, not just at the Guard layer.
- No string interpolation in raw SQL — use Prisma's typed query API or `$queryRaw` with `Prisma.sql` tagged templates exclusively.
- No secrets in source code, comments, or committed `.env` files. Access all secrets via `@dev-libraries` `ConfigService` wrapper.
- Never return internal error messages, stack traces, or system paths in API responses. Use a global `ExceptionFilter` to shape all error output.
- Strip or mask PII from logs before they reach any external sink. Use `@dev-libraries/logger` — never `console.*`.
- `@Public()` decorator (or equivalent bypass) must be explicitly justified with a comment at the route.

---

## TypeScript

- Prefer `interface` for object shapes; `type` for unions, intersections, and mapped types.
- Never use non-null assertion (`!`) unless a preceding guard makes it provably safe.
- No `any`. Use `unknown` and narrow explicitly.
- Export types and interfaces from the module that owns them; re-export from `index.ts` if needed.

---

## Module Structure

> Pinned: **NestJS 10.x**.

- One feature per module. Modules encapsulate their own Controller, Service, and any Kafka consumer/producer concerns.
- Never import a Service from another module directly — expose it via that module's `exports` array and import the module.
- `AppModule` imports only feature modules and global infrastructure modules. No business logic in `AppModule`.
- Shared utilities (guards, interceptors, pipes, decorators) live in `common/` or `shared/`. Never duplicated across features.
- Circular module dependencies are a design smell — resolve by extracting a shared module or inverting the dependency.

---

## Controllers

- Controllers are thin routing layers only: parse input, delegate to Service, return result. No business logic.
- All routes must have explicit `@ApiTags`, `@ApiOperation`, and `@ApiResponse` decorators (Swagger-first).
- All DTOs used in request bodies, query params, and responses must be explicitly typed — no `Record<string, any>`.
- Use `@HttpCode` explicitly when the default 200/201 is incorrect.
- Route parameters that identify resources (`id`, `slug`) must be validated at the param level (`ParseUUIDPipe`, `ParseIntPipe`).

---

## DTOs & Validation

- Separate `CreateXDto`, `UpdateXDto` (`PartialType` of Create), and `ResponseXDto` — never reuse input DTOs for output.
- `ResponseXDto` must use `@Exclude()` / `@Expose()` with `ClassSerializerInterceptor` to prevent field leakage. Never return Prisma model objects directly from Controllers.
- `UpdateXDto` must be `PartialType(CreateXDto)`, not a hand-rolled partial.

---

## Services

- Services own all business logic, orchestration, and transaction management.
- A Service must not depend on `Request` or any HTTP-layer object. It must be usable outside HTTP context (queue consumers, CLI, Kafka processors).
- Throw domain exceptions (`NotFoundException`, `ForbiddenException`, `ConflictException`) from Services — never return `null` and leave the caller to handle ambiguity.
- For mutations spanning multiple Prisma operations, use `prisma.$transaction([...])` (sequential) or the interactive transaction callback (`prisma.$transaction(async (tx) => {...})`) for logic-dependent flows. Pass `tx` as a parameter — never re-inject `PrismaService` inside a transaction callback.

---

## Data Access (Prisma)

- `PrismaService` (from `@dev-libraries/db`) is the single DB access point. Inject it directly into Services. No wrapper repositories.
- Never access `PrismaService` from Controllers, Guards, or Interceptors.
- Prisma model types (`Prisma.XGetPayload`, `Prisma.XSelect`) are internal to the Service layer. Map to DTOs before returning across any boundary.
- Raw queries (`$queryRaw`) require an explicit comment justifying why the Prisma query API is insufficient. Always use `Prisma.sql` tagged templates — never string interpolation.
- Pagination: always apply `take`/`skip` or cursor-based limits. Never return unbounded collections. Default page size must be explicit, not inferred.
- `select` and `include` must be scoped to only the fields needed — never use implicit full-model fetches for list endpoints.

---

## Authentication & Authorization

- Use `@dev-libraries/auth` exclusively for JWT issuance, verification, refresh token logic, and session handling. Never implement these inline.
- Auth is enforced via Guards. A `JwtAuthGuard` backed by `@dev-libraries/auth` must be set globally; opt-out with `@Public()` (justified with a comment).
- Role/permission checks use a dedicated `RolesGuard` or `AbilityGuard`. Never inline role logic in Controllers or Services.
- `request.user` is the single source of identity truth inside a request lifecycle. Never accept identity from request body or query string.

---

## Rate Limiting

- Use `@dev-libraries/rate-limiter` exclusively. Never configure `@nestjs/throttler` or any other limiter directly.
- Apply rate limits at the route level for public/unauthenticated endpoints and at the module level for authenticated APIs. Document the chosen limit and rationale in a comment at the decorator site.
- Bypass or custom limits (e.g. internal service calls) require explicit justification and must not be applied globally.

---

## Redis

- Use `@dev-libraries/redis` exclusively for all Redis interactions. Never instantiate `ioredis` or any Redis client directly.
- Permitted uses: caching, distributed locking, session storage, rate-limiter backing store, pub/sub.
- All cache keys must follow a namespaced convention: `<service>:<entity>:<id>` (e.g. `auth:session:uuid`). Document key schema at the point of first use.
- Always set an explicit TTL on every key. Never persist without expiry unless justified in a comment.
- Cache-aside pattern: read cache → on miss, read DB → write cache. Never write to cache inside a Prisma transaction.

---

## Kafka

- Use `@dev-libraries/kafka` exclusively for producer and consumer setup. Never configure `kafkajs` directly.
- Producers live in the Service layer. Never produce events from Controllers or Guards.
- Consumers (`@MessagePattern` / `@EventPattern`) are registered in a dedicated module. Consumer classes follow the same rules as Services: no HTTP dependencies, explicit error handling.
- Every Kafka message payload must have a typed DTO validated on consumption — treat incoming Kafka messages as untrusted external input.
- Idempotency: every consumer must document its idempotency strategy (deduplication key, at-least-once tolerance, etc.) as a comment at the class level.
- Dead-letter strategy must be defined at the consumer class level. Unhandled exceptions in a consumer must not silently drop messages.
- Never produce events inside a `prisma.$transaction` callback — Kafka and DB commits are not atomic. Produce after confirmed DB commit, or use the outbox pattern (document which applies).

---

## AWS SDK

- Use `@dev-libraries/aws` exclusively. Never import `@aws-sdk/*` directly.
- S3 operations: always specify bucket and key explicitly. Never construct keys via string interpolation from user input.
- SQS: treat all incoming message bodies as untrusted — validate with `class-validator` DTO before processing.
- Signed URLs: set the shortest viable expiry. Document the expiry rationale at the call site.
- Never log full AWS responses — they may contain presigned URLs or credentials.

---

## Metrics

- Use `@dev-libraries/metrics` exclusively. Never import `prom-client` directly.
- Instrument at these points as a minimum: HTTP request duration (via global interceptor), Kafka consumer lag, Prisma query duration for slow queries (>200ms), and AWS SDK call failures.
- Metric names follow `snake_case` with a `<service>_` prefix. Document unit (seconds, bytes, count) at the metric definition.
- Never record PII or user-identifiable values as metric labels.

---

## Logger

- Use `@dev-libraries/logger` exclusively. Never use `console.*`, `winston`, or `pino` directly.
- Inject `Logger` with the class name as context: `new Logger(MyService.name)`.
- Log levels: `error` for caught exceptions before re-throw; `warn` for recoverable degraded states; `log` for significant lifecycle events; `debug` for dev-only tracing (must not appear in production builds).
- Never log raw request bodies, JWT tokens, passwords, PII, or AWS credentials.
- Structured log fields: always include `traceId` (from request context) for request-scoped logs.

---

## Configuration & Environment

- All env vars are accessed exclusively via `@dev-libraries` `ConfigService`. Never access `process.env` directly outside of that library's internals.
- Secrets must never appear in config defaults or fallback values.
- Feature flags and environment-specific behavior are driven by `ConfigService`, not scattered `process.env` checks.

---

## Error Handling

- Catch errors at the boundary closest to the failure.
- Log the full error (including stack) with the injected `Logger` before any re-throw.
- Re-throw unexpected errors as a domain `InternalServerErrorException` with a caller-safe message. Never surface raw Prisma errors, stack traces, or system paths.
- Use a global `ExceptionFilter` to normalize all unhandled exceptions into a consistent `{ statusCode, message, timestamp, path }` shape. Prisma `PrismaClientKnownRequestError` codes (e.g. `P2002` unique constraint, `P2025` not found) must be mapped to appropriate HTTP exceptions inside this filter or in the Service.
- Validation errors from `ValidationPipe` resolve to 400 automatically. Do not catch and re-wrap them.

---

## Async, Queues & Scheduling

- CPU-bound or long-running work must be offloaded to a Kafka consumer or BullMQ worker. Never block the event loop inside a request handler.
- Use `@nestjs/schedule` for cron jobs via `@dev-libraries` configuration. Document timezone, expected duration, and idempotency guarantee at the decorator site.
- Scheduled jobs that perform DB writes must be idempotent — document the mechanism (e.g. upsert, deduplication key).

---

## Interceptors, Pipes & Middleware

- `ClassSerializerInterceptor` and `ValidationPipe` are global. Do not re-declare at controller or route level unless overriding defaults — document the override reason.
- Middleware is for cross-cutting concerns with no business logic: correlation ID injection, request logging. Never put auth or authorization logic in middleware.
- Scope interceptors and guards to the narrowest applicable level (global → module → controller → route). Prefer global for enforcement, narrow for exceptions.

---

## Testing

- Unit tests (`*.spec.ts`) mock all dependencies via `Test.createTestingModule`. Mock `@dev-libraries` modules at the provider level — never mock their internals.
- Integration tests (`*.e2e-spec.ts`) use a Testcontainer with a real Postgres instance. Never use the production DB. Run Prisma migrations against the test container before the suite.
- Every Guard, Interceptor, and Pipe must have standalone unit tests.
- Service tests must cover at minimum: happy path, not-found, forbidden, conflict, and transaction-failure branches.
- Kafka consumer tests must cover: valid payload, invalid payload (validation rejection), and idempotent re-delivery.
