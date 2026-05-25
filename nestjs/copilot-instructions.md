# Copilot Workspace Instructions

Applies to: test generation, JSDoc authoring, and class-validator DTO authoring.

---

## General

- Never generate code that violates the Project Rules (`/.github/project-rules.md`).
- Never introduce a dependency not present in `package.json`. If a pattern requires one, leave a `// TODO:` comment and stop.
- Never auto-apply changes. Present as a diff.
- If the correct approach is ambiguous, emit a `// COPILOT: ambiguous —` comment describing the ambiguity and stop.

---

## Test Generation

### Scope

Applies to all `*.spec.ts` files under `src/`.

### Rules

- Use `Test.createTestingModule` for all NestJS unit tests. Never instantiate classes directly.
- Mock every injected dependency. Use `jest.fn()` for methods; type mocks as `jest.Mocked<T>`.
- Never make real DB, Redis, Kafka, AWS, or HTTP calls in `*.spec.ts`. Mock the corresponding infra provider (`PrismaService`, `RedisService`, `KafkaProducerService`, `AwsService`).
- Never mock `@dev-libraries` internals — mock the `infra/` wrapper provider that exposes it.
- Test file lives alongside its subject. Never create a new directory for tests.

### Required Coverage per Service

Every generated Service spec must include, at minimum:

| Branch                        | Assertion                             |
| ----------------------------- | ------------------------------------- |
| Happy path                    | Returns expected mapped DTO           |
| Not found                     | Throws `NotFoundException`            |
| Forbidden / ownership failure | Throws `ForbiddenException`           |
| Conflict (unique violation)   | Throws `ConflictException`            |
| Prisma transaction failure    | Throws `InternalServerErrorException` |

### Required Coverage per Kafka Consumer

| Branch                               | Assertion                                                         |
| ------------------------------------ | ----------------------------------------------------------------- |
| Valid payload                        | Delegates to Service with correct args                            |
| Invalid payload (validation failure) | Does not call Service; does not throw silently                    |
| Idempotent re-delivery               | Documents expected behavior in a `it.todo` if not yet implemented |

### Structure

```ts
describe('<ClassName>', () => {
  let service: MyService;
  let prisma: jest.Mocked<PrismaService>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        MyService,
        { provide: PrismaService, useValue: { user: { findUnique: jest.fn() } } },
      ],
    }).compile();

    service = module.get(MyService);
    prisma = module.get(PrismaService);
  });

  afterEach(() => jest.clearAllMocks());

  describe('<methodName>', () => {
    it('should ...', async () => { ... });
    it('should throw NotFoundException when ...', async () => { ... });
  });
});
```

- One top-level `describe` per class.
- One nested `describe` per public method.
- `it` descriptions start with `"should"` and describe observable behavior, not implementation.
- `afterEach(() => jest.clearAllMocks())` is always present.
- No `beforeAll` for stateful setup — use `beforeEach` to keep tests isolated.

---

## JSDoc Authoring

### Scope

Applies to all files under `src/`. Priority order: Services → Controllers → infra providers → DTOs → guards/interceptors.

### Rules

- Document **intent and contracts**, not implementation. Never restate what the code visibly does.
- Every `@param` must describe the semantic meaning, not the type (the type is already in the signature).
- Every `@throws` must name the exception class and the condition that triggers it.
- Every `@remarks` block that references a `@dev-libraries` package must name the package.

### Required Tags by Construct

**Service method:**

```ts
/**
 * <One sentence: what it does and why it exists.>
 *
 * @param id - <Semantic meaning, e.g. "UUID of the resource the caller owns">
 * @returns <What the resolved value represents>
 * @throws {NotFoundException} <Condition>
 * @throws {ForbiddenException} <Condition>
 * @remarks
 * Produces a `<event.name>` Kafka event via `@dev-libraries/kafka` after commit.
 * Uses `prisma.$transaction` — callers must not wrap this in an outer transaction.
 */
```

**Controller route:**

```ts
/**
 * <One sentence: HTTP contract summary.>
 *
 * @remarks
 * Rate-limited via `@dev-libraries/rate-limiter`. Requires JWT. Role: <role>.
 */
```

**Kafka Consumer class:**

```ts
/**
 * Consumes `<topic>` events and delegates to `<ServiceName>`.
 *
 * @remarks
 * Idempotency: <describe deduplication key or at-least-once tolerance>.
 * Dead-letter strategy: <describe>.
 */
```

**infra provider method:**

```ts
/**
 * <One sentence: what operation this exposes from @dev-libraries/<package>.>
 *
 * @remarks
 * Thin wrapper — do not add business logic here.
 */
```

### Anti-patterns — never generate these

```ts
// ❌ Restates the code
/** Finds a user by ID and returns it. */
async findById(id: string) { ... }

// ❌ Type-repeating param
/** @param id - string */

// ❌ Vague throws
/** @throws Error */
```

---

## Class-Validator DTO Authoring

### Scope

Applies to all `*.dto.ts` files under `src/modules/`.

### Rules

- Every DTO property must have at least one `class-validator` decorator. No bare typed properties.
- Every DTO class must have `@ApiProperty` or `@ApiPropertyOptional` on every property (Swagger-first).
- `UpdateXDto` must be `PartialType(CreateXDto)`. Never hand-roll a partial.
- `ResponseXDto` must mark sensitive or internal fields with `@Exclude()`. Apply `@Expose()` only on fields explicitly intended for output.
- Never use `@IsOptional()` on a `CreateXDto` field unless the field is genuinely optional at creation — use `UpdateXDto` for partial updates.
- Nested objects must use `@ValidateNested()` + `@Type(() => NestedDto)`. Never validate nested shapes with `@IsObject()` alone.
- Arrays must use `@IsArray()` + `@ArrayMinSize(1)` (or justified minimum) + item-level decorator via `@Type`.
- Enum properties must use `@IsEnum(MyEnum)` — never `@IsString()` with a comment.

### Required Structure

**CreateXDto:**

```ts
import {
  IsString,
  IsUUID,
  IsEnum,
  IsNotEmpty,
  MaxLength,
} from "class-validator";
import { ApiProperty } from "@nestjs/swagger";
import { MyEnum } from "../types/my.enum";

export class CreateMyResourceDto {
  @ApiProperty({
    description: "<semantic meaning>",
    example: "<realistic example>",
  })
  @IsString()
  @IsNotEmpty()
  @MaxLength(255)
  name: string;

  @ApiProperty({ enum: MyEnum, description: "<semantic meaning>" })
  @IsEnum(MyEnum)
  status: MyEnum;
}
```

**UpdateXDto:**

```ts
import { PartialType } from "@nestjs/swagger";
import { CreateMyResourceDto } from "./create-my-resource.dto";

export class UpdateMyResourceDto extends PartialType(CreateMyResourceDto) {}
```

**ResponseXDto:**

```ts
import { Exclude, Expose } from "class-transformer";
import { ApiProperty } from "@nestjs/swagger";

@Exclude()
export class MyResourceResponseDto {
  @Expose()
  @ApiProperty()
  id: string;

  @Expose()
  @ApiProperty()
  name: string;

  // internalField is excluded by @Exclude() on the class — not listed here
}
```

### Anti-patterns — never generate these

```ts
// ❌ Bare property
name: string;

// ❌ IsObject() for nested shape
@IsObject()
address: AddressDto;

// ❌ IsString() for enum
@IsString()
status: 'active' | 'inactive';

// ❌ Hand-rolled partial
export class UpdateDto { name?: string; }
```
