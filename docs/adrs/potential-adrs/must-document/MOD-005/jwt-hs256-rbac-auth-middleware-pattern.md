# JWT HS256 + RBAC Auth Middleware Pattern

## Score
Total: 146/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 23/25 | Team Knowledge: 23/25

## Evidence
- File: `src/middlewares/auth.middleware.ts` — `authenticate` verifies `Authorization: Bearer` header using `jwt.verify` with `env.JWT_SECRET`; attaches `AuthUser { id, email, role }` to `req.user`; `requireRole(...roles)` checks `req.user.role` and throws `ForbiddenError`
- File: `src/config/env.ts` — `JWT_SECRET` enforced minimum 16 chars; `JWT_EXPIRES_IN` defaults to `8h`
- File: `prisma/schema.prisma` — `enum UserRole { ADMIN, OPERATOR }` encoded at schema level
- Transcript: `[09:35] Sofia` — DLQ replay endpoint requires ADMIN role
- Transcript: `[09:36] Larissa` — "`requireRole('ADMIN')` já existe no projeto. A gente reutiliza."

## Decision Summary
Authentication uses stateless JWT HS256 tokens with `{ sub, email, role }` claims. Two composable Express middleware factories handle identity and authorization separately: `authenticate` (verifies token, attaches `AuthUser` to `req.user`) and `requireRole(...roles)` (checks role against the allowed set, throws `ForbiddenError` if unauthorized). Routers apply `authenticate` at the router level via `router.use(authenticate)`, then add `requireRole` on individual routes that need role restriction. The two-role model (`ADMIN`, `OPERATOR`) is encoded in the Prisma schema enum.

## Context
The API is an internal B2B system with a controlled user base. Stateless JWT tokens eliminate the need for a session store (Redis, DB table) and simplify horizontal scaling. Two roles are sufficient for the current permission model; the `requireRole` factory accepts a spread of roles, allowing future expansion without breaking the middleware contract.

## Alternatives Considered
- **Session-based auth with server-side store:** Requires Redis or a DB session table; breaks stateless horizontal scaling; more complex for the current team size and scale.
- **OAuth2/OIDC external provider:** Overkill for an internal B2B API with a small, controlled user base; adds an external dependency and operational complexity.
- **Single "authenticated or not" check (no RBAC):** Insufficient — admin-only operations (user management, DLQ replay) require role enforcement.

## Consequences
**Positive:**
- Stateless tokens: no session store needed; horizontal scaling is trivial
- `requireRole('ADMIN')` is reusable for any future admin-only endpoint (e.g., DLQ replay per [09:36] Larissa)
- Role is encoded in the token — no DB lookup on every request
- Separation of `authenticate` and `requireRole` allows per-route granularity

**Negative / Trade-offs:**
- Tokens cannot be invalidated before expiry without a token denylist (not implemented); a leaked token remains valid until `JWT_EXPIRES_IN`
- `JWT_SECRET` is a symmetric key — compromise of the secret allows forging any token
- `8h` default expiry may be too long for high-security contexts

## Related Modules
- MOD-002 (Auth) — token issuance
- MOD-003 (Users) — `requireRole('ADMIN')` on user management endpoint
- MOD-008 (Webhooks) — `requireRole('ADMIN')` on DLQ replay endpoint; `authenticate` on all webhook CRUD routes
