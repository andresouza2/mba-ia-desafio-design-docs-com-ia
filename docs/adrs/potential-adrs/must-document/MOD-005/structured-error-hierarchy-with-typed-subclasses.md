# Structured Error Hierarchy with Typed Subclasses

## Score
Total: 145/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 22/25 | Team Knowledge: 23/25

## Evidence
- File: `src/shared/errors/app-error.ts` — `AppError` base class with `statusCode`, `errorCode` (string constant), optional `details`; `Error.captureStackTrace` for clean stacks
- File: `src/shared/errors/http-errors.ts` — 9 concrete subclasses: `BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, `InsufficientStockError`; domain errors carry typed `details`
- File: `src/middlewares/error.middleware.ts` — primary branch is `instanceof AppError`
- Transcript: `[09:28] Bruno` — "os erros usam o mesmo AppError, só muda o prefixo. WEBHOOK_ alguma coisa."
- Transcript: `[09:29] Larissa` — "Prefixo WEBHOOK_ pra tudo do módulo"

## Decision Summary
All application errors extend a single `AppError` base class that carries `statusCode`, `errorCode` (a `SCREAMING_SNAKE_CASE` string), and optional `details`. Domain-specific subclasses (e.g., `InsufficientStockError`, `InvalidStatusTransitionError`) extend this base with domain-prefixed error codes and typed `details` payloads. The centralized error middleware identifies any error as `instanceof AppError` and formats it uniformly. New modules extend the hierarchy with their own prefix (e.g., `WEBHOOK_NOT_FOUND`) without modifying the middleware.

## Context
Without a structured hierarchy, different modules would throw plain `Error` objects or `{ status, message }` ad-hoc objects, requiring the error middleware to handle dozens of unrelated shapes. The typed hierarchy makes error formatting deterministic and extensible.

## Alternatives Considered
- **Plain `Error` with duck-typed properties:** No TypeScript type-safety; middleware cannot reliably distinguish domain errors from unexpected runtime errors.
- **Single generic `ApiError(status, code)` factory:** Cannot carry typed domain-specific `details` (e.g., `{ sku, requested, available }` for `InsufficientStockError`).
- **Result/Either types:** Requires all callers to pattern-match on results; adds boilerplate to controllers without meaningful benefit in an Express app where errors naturally propagate via `next(err)`.

## Consequences
**Positive:**
- New modules add `WEBHOOK_*` error subclasses with zero changes to the middleware
- TypeScript enforces that `errorCode` and `details` are set correctly at construction time
- `instanceof AppError` is a single reliable check in the error middleware

**Negative / Trade-offs:**
- Error code strings (`SCREAMING_SNAKE_CASE`) are not enforced by the type system — convention must be documented and enforced via code review
- As the number of subclasses grows, `http-errors.ts` becomes a large file; may eventually need splitting by domain

## Related Modules
- All modules (MOD-001 through MOD-008) — every module throws `AppError` subclasses
