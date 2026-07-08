# Centralized Error Middleware (Multi-Type Catch)

## Score
Total: 142/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 20/25 | Team Knowledge: 22/25

## Evidence
- File: `src/middlewares/error.middleware.ts` — single Express 4-argument error handler catching `AppError`, `ZodError` (with field-level formatting), and `Prisma.PrismaClientKnownRequestError` (P2002 → 409, P2025 → 404); unhandled errors logged via Pino and returned as 500
- File: `src/middlewares/validate.middleware.ts` — converts `ZodError` to `ValidationError` before reaching the error middleware (double-layer defence)
- Transcript: `[09:29] Bruno` — "O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada."

## Decision Summary
A single Express 4-argument error handler (`errorMiddleware`) at the application root catches all errors and formats them into a uniform `{ error: { code, message, details } }` JSON shape. It handles three error categories: `AppError` subclasses (domain errors), `ZodError` (validation failures at the middleware level), and `Prisma.PrismaClientKnownRequestError` (ORM constraint violations translated to HTTP semantics — P2002 → 409 Conflict, P2025 → 404 Not Found). All other errors are logged and returned as 500 without leaking internal details.

## Context
Without a centralized handler, each controller would need try/catch blocks with individual formatting logic. Prisma constraint errors would surface as 500s unless explicitly caught everywhere. The centralized middleware ensures consistent error shapes across all modules and prevents ORM internals from leaking to API clients.

## Alternatives Considered
- **Per-route try/catch with individual formatting:** Significant duplication; inconsistent error shapes across modules; Prisma errors easily missed.
- **Multiple error middlewares registered by type:** Express executes error middlewares in order; type-based registration creates ordering sensitivity and is harder to reason about than a single handler with explicit type checks.
- **Letting Prisma errors propagate as unhandled 500s:** Leaks ORM error codes and stack traces to clients; P2002 unique constraint violations should be 409, not 500.

## Consequences
**Positive:**
- Uniform error response shape across all endpoints; clients have a single contract to handle
- Prisma constraint errors are translated to meaningful HTTP semantics without per-repository boilerplate
- Webhook module errors (`WEBHOOK_*`) propagate automatically — no middleware changes required
- Unhandled errors are logged with full context before returning a sanitized 500

**Negative / Trade-offs:**
- The error middleware must be kept up to date as new Prisma error codes (P20xx) become relevant
- The double-layer (`validate` middleware converts `ZodError` → `ValidationError`, then the error middleware also handles raw `ZodError`) must be documented to avoid confusion

## Related Modules
- All modules with HTTP routes (MOD-001 through MOD-008)
