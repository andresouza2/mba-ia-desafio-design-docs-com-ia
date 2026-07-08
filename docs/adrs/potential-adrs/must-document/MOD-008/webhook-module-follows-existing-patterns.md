# Webhook Module Follows Existing Controller→Service→Repository Pattern and Reuses Shared Infrastructure

## Score
Total: 125/150 | Step 0: 75 | Scope+Impact: 20/25 | Cost to Change: 15/25 | Team Knowledge: 15/25

## Evidence
- Transcript: `[09:27] Bruno` — "a gente não precisa inventar nada novo. Segue o mesmo padrão dos outros módulos."
- Transcript: `[09:28] Diego` — "controller, service, repository, schemas. Igual."
- Transcript: `[09:28] Bruno` — "os erros usam o mesmo AppError, só muda o prefixo. WEBHOOK_ alguma coisa."
- Transcript: `[09:29] Diego` — "o logger é o mesmo Pino. O error middleware já trata AppError, não precisa mudar."
- File: `src/shared/errors/app-error.ts` — `AppError` base class with `statusCode`, `errorCode`, `details` — extensible for WEBHOOK_ codes
- File: `src/middlewares/error.middleware.ts` — already handles all `AppError` subclasses generically
- File: `src/app.ts` — `buildControllers(prisma)` is the composition root where the webhook controller will be wired

## Decision Summary
The Webhooks module (MOD-008) follows the identical structural pattern of all other modules: `webhook.controller.ts` → `webhook.service.ts` → `webhook.repository.ts` + `webhook.routes.ts` + `webhook.schemas.ts`. It reuses `AppError` (new subclasses with `WEBHOOK_` error code prefix), the centralized `errorMiddleware`, `authenticate` and `requireRole` middlewares, `validate` middleware with Zod schemas, and the Pino `logger`. No new frameworks, IoC containers, or infrastructure patterns are introduced.

## Context
The team explicitly decided against introducing new architectural patterns for the webhook module. Consistency lowers the learning curve, makes code reviews faster, and ensures the error middleware, logging, and validation middleware all work without modification. The `WEBHOOK_` prefix distinguishes webhook-specific error codes from existing codes while staying within the established `errorCode` string convention.

## Alternatives Considered
- **Introducing an IoC container or DI framework:** Rejected — the existing manual DI via `buildControllers(prisma)` works well for the team's scale and adding a framework would require refactoring all existing modules. ([09:27] Bruno)
- **Event emitter / Observer pattern instead of direct function call:** Not discussed for internal module structure; the team stayed with direct function calls following existing patterns.

## Consequences
**Positive:**
- Zero new patterns to learn — any developer familiar with the Orders module can immediately read and contribute to Webhooks
- `errorMiddleware` requires no changes; `WEBHOOK_*` error subclasses propagate automatically
- Validation, authentication, and logging middleware apply without modification
- `buildControllers(prisma)` composition root only needs the new controller added

**Negative / Trade-offs:**
- Controller→Service→Repository may feel over-engineered for simpler webhook admin endpoints
- `WEBHOOK_` prefix convention must be documented to prevent ad-hoc error code naming

## Related Modules
- MOD-005 (Shared) — AppError, errorMiddleware, authenticate, requireRole, validate, logger
- MOD-001 (Infrastructure) — `buildControllers(prisma)` wires the new webhook controller
- All existing modules — establishes the pattern being replicated
