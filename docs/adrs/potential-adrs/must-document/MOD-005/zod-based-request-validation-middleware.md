# Zod-Based Request Validation Middleware

## Score
Total: 138/150 | Step 0: 75 | Scope+Impact: 23/25 | Cost to Change: 20/25 | Team Knowledge: 20/25

## Evidence
- File: `src/middlewares/validate.middleware.ts` — `validate({ body?, query?, params? })` factory returns an Express `RequestHandler`; parses each request part with the provided Zod schema; wraps `ZodError` into `ValidationError` with field-level `path`/`message` details
- All domain modules have `*.schemas.ts` files with Zod schemas (e.g., `createOrderSchema`, `listOrdersQuerySchema` with coerced pagination, `orderIdParamSchema`)
- File: `src/config/env.ts` — Zod used for environment validation at startup — same library, different use case
- Mapping: planned webhook schemas will use `z.string().url()` with `https` enforcement, `z.array(z.nativeEnum(OrderStatus))` for event type filtering

## Decision Summary
All HTTP request input (body, query string, path params) is validated using Zod schemas defined per module in `*.schemas.ts` files. The shared `validate({ body?, query?, params? })` middleware factory applies these schemas before the request reaches the controller and converts `ZodError` into a typed `ValidationError` (400) with per-field error details. Zod is also reused for environment variable validation at startup, making it the single validation library in the project.

## Context
Without request validation, controllers would receive untyped, potentially malformed data requiring defensive checks throughout the business logic layer. Centralizing validation in the middleware layer keeps controllers clean and ensures all input is type-safe before it reaches business logic. Reusing Zod for both HTTP and env validation eliminates a dependency on a second validation library.

## Alternatives Considered
- **express-validator:** Less type-safe; more imperative; requires field-by-field validation chain syntax.
- **Joi:** Mature but no TypeScript type inference from schemas; inferred types require manual type annotations.
- **Hand-written validation in controllers:** Duplication across modules; inconsistent error shapes; no reusable schema objects.
- **class-validator with class-transformer:** Requires decorators (`experimentalDecorators`), adds a transformation layer, and is more complex than Zod for simple request schemas.

## Consequences
**Positive:**
- TypeScript types are inferred directly from Zod schemas — no duplicate type declarations
- `validate()` middleware is a single, reusable factory applied consistently across all routes
- Validation errors have a consistent per-field format (`path`, `message`) clients can rely on
- Zod's `.coerce` is used for query string pagination params — handles the string-to-number conversion transparently

**Negative / Trade-offs:**
- Zod validation runs synchronously on every request; for deeply nested schemas with many fields, this adds measurable (though usually negligible) latency
- Schema files grow as modules add more endpoints; large schemas should be split by operation

## Related Modules
- All domain modules (MOD-002 through MOD-008) — each defines Zod schemas consumed via `validate()`
- MOD-001 (Infrastructure) — `src/config/env.ts` uses Zod for startup validation
