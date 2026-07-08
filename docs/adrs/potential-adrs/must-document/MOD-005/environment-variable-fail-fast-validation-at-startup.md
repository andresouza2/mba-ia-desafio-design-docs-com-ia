# Environment Variable Fail-Fast Validation at Startup

## Score
Total: 122/150 | Step 0: 75 | Scope+Impact: 20/25 | Cost to Change: 12/25 | Team Knowledge: 15/25

## Evidence
- File: `src/config/env.ts` — `envSchema` (Zod) validates all env vars at module load; `loadEnv()` calls `envSchema.safeParse(process.env)`, prints all invalid fields with paths and messages, then calls `process.exit(1)` on failure; `env` exported as frozen parsed object
- File: `src/shared/logger/index.ts` — imports `env.LOG_LEVEL` at module level; guaranteed valid at runtime
- File: `src/middlewares/auth.middleware.ts` — imports `env.JWT_SECRET` at module level; no runtime null-check needed

## Decision Summary
All environment variables are validated against a Zod schema at application startup, before any server socket is opened. If any required variable is missing or fails its constraint (e.g., `JWT_SECRET` shorter than 16 characters), the process prints human-readable diagnostics for every failing field and exits with code 1. The parsed, type-safe `env` object is the sole source of configuration for the entire application — no `process.env` access occurs outside `src/config/env.ts`.

## Context
Misconfigured deployments (missing `DATABASE_URL`, short `JWT_SECRET`) should fail immediately with a clear diagnostic rather than failing at the first request or — worse — silently using a default that creates a security vulnerability. The fail-fast pattern surfaces configuration errors at deploy time, not at runtime under production load.

## Alternatives Considered
- **Lazy validation at first use of each env var:** Configuration errors surface deep in code paths, often as cryptic runtime errors rather than clear "missing env var" messages; difficult to diagnose in production.
- **dotenv with manual `process.env.X || 'default'` across files:** No centralized schema; defaults are inconsistent; no validation of format constraints (e.g., minimum length for secrets).
- **No validation:** Misconfigured deployments fail silently or with unrelated error messages; security-sensitive defaults (e.g., `JWT_SECRET = ''`) go undetected.

## Consequences
**Positive:**
- Configuration errors are caught at deployment time with actionable diagnostics per field
- `env` is a frozen, typed object — downstream code never needs null-checks or defaults
- `JWT_SECRET` minimum-length constraint prevents weak secrets from being deployed
- Worker process will import the same `env` module, getting identical validation guarantees

**Negative / Trade-offs:**
- Adding a new required env var immediately breaks existing deployments that haven't set it; requires coordinated infrastructure change
- `process.exit(1)` in a module makes `src/config/env.ts` harder to import in unit tests without mocking

## Related Modules
- MOD-001 (Infrastructure) — `server.ts` imports `env.PORT`
- MOD-005 (Shared) — `logger` imports `env.LOG_LEVEL`; auth middleware imports `env.JWT_SECRET`
- MOD-008 (Webhooks) — worker process imports `env.DATABASE_URL`, `env.LOG_LEVEL`, `env.NODE_ENV`
