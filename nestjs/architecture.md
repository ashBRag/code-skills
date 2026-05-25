# Folder Structure

```
src/
├── main.ts                          # Bootstrap, global pipes/filters/interceptors
├── app.module.ts                    # Root module — imports only feature + infra modules
│
├── config/
│   └── env.validation.ts            # @dev-libraries ConfigService schema
│
├── common/
│   ├── decorators/
│   │   ├── public.decorator.ts      # @Public() bypass marker
│   │   └── roles.decorator.ts       # @Roles() metadata setter
│   ├── filters/
│   │   └── exception.filter.ts      # Global ExceptionFilter — maps Prisma + domain errors
│   ├── guards/
│   │   ├── jwt-auth.guard.ts        # Wraps @dev-libraries/auth JWT verification
│   │   └── roles.guard.ts           # Role/permission enforcement
│   ├── interceptors/
│   │   ├── serialize.interceptor.ts # ClassSerializerInterceptor wrapper
│   │   ├── metrics.interceptor.ts   # HTTP request duration via @dev-libraries/metrics
│   │   └── trace.interceptor.ts     # Injects traceId into logger context
│   ├── pipes/
│   │   └── validation.pipe.ts       # Global ValidationPipe config
│   └── dto/
│       └── pagination.dto.ts        # Shared take/skip + cursor DTOs
│
├── modules/
│   ├── <feature>/
│   │   ├── <feature>.module.ts
│   │   ├── <feature>.controller.ts
│   │   ├── <feature>.service.ts
│   │   ├── dto/
│   │   │   ├── create-<feature>.dto.ts
│   │   │   ├── update-<feature>.dto.ts
│   │   │   └── response-<feature>.dto.ts
│   │   ├── events/
│   │   │   ├── <feature>.producer.ts     # Kafka producer — called by Service post-commit
│   │   │   └── <feature>.consumer.ts     # @EventPattern handler — validates payload, delegates to Service
│   │   ├── <feature>.controller.spec.ts
│   │   └── <feature>.service.spec.ts
│   │
│   └── health/
│       ├── health.module.ts
│       └── health.controller.ts          # /health — liveness + readiness probes
│
├── infra/                                # Thin wrappers that expose @dev-libraries packages as NestJS providers
│   ├── prisma/
│   │   ├── prisma.module.ts              # Global module — exports PrismaService
│   │   └── prisma.service.ts             # Wraps @dev-libraries/db, handles onModuleInit/onModuleDestroy
│   ├── redis/
│   │   ├── redis.module.ts               # Global module — exports RedisService
│   │   └── redis.service.ts              # Wraps @dev-libraries/redis
│   ├── kafka/
│   │   ├── kafka.module.ts               # Global module — exports KafkaProducerService
│   │   └── kafka-producer.service.ts     # Wraps @dev-libraries/kafka producer
│   ├── aws/
│   │   ├── aws.module.ts
│   │   └── aws.service.ts                # Wraps @dev-libraries/aws — S3, SQS facades
│   ├── metrics/
│   │   ├── metrics.module.ts
│   │   └── metrics.service.ts            # Wraps @dev-libraries/metrics — exposes meter/counter/histogram
│   └── logger/
│       ├── logger.module.ts
│       └── logger.service.ts             # Wraps @dev-libraries/logger — provides Logger globally
│
└── types/
    └── express.d.ts                      # Module augmentation — extends Request with user identity type
```

## Placement Rules

- **Feature modules** go in `modules/<feature>/`. One directory per aggregate root.
- **Kafka producers** are co-located with the feature that owns the event. **Consumers** are also co-located with the feature that processes the event — not necessarily the one that produces it.
- **`infra/`** contains only provider wrappers around `@dev-libraries`. No business logic. Each infra module is `@Global()`.
- **`common/`** contains only framework-level cross-cutting concerns (guards, filters, interceptors, pipes, shared decorators). Nothing in `common/` imports from `modules/`.
- **`common/dto/`** is for DTOs shared across two or more features (e.g. pagination). Feature-specific DTOs stay inside `modules/<feature>/dto/`.
- **Tests** are co-located with their subject (`*.spec.ts` alongside the source file). E2E tests live in `test/` at the project root.

## What Goes Where — Quick Reference

| Concern                   | Location                                                               |
| ------------------------- | ---------------------------------------------------------------------- |
| JWT verification, session | `common/guards/jwt-auth.guard.ts` via `@dev-libraries/auth`            |
| Rate limiting             | Applied at route/module via `@dev-libraries/rate-limiter` decorator    |
| Prisma access             | `PrismaService` injected into `modules/<feature>/<feature>.service.ts` |
| Cache (Redis)             | `RedisService` injected into Service layer only                        |
| Kafka produce             | `<feature>.producer.ts` called by Service after confirmed DB commit    |
| Kafka consume             | `<feature>.consumer.ts` validates payload DTO, delegates to Service    |
| AWS S3/SQS                | `AwsService` injected into Service layer only                          |
