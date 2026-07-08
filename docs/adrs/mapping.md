# Codebase Mapping

## Project Overview

- **Name:** order-management-api
- **Purpose:** Order Management System (OMS) — REST API for managing customers, products, orders, and users, with a controlled order-lifecycle state machine, transactional stock management, and audit history. The system is the target for a new outbound Webhook Notification feature that will push order-status-change events to B2B customer endpoints.
- **Tech Stack Summary:** Node.js 20 + TypeScript 5.6, Express 4, Prisma ORM 5 over MySQL 8, JWT authentication, Zod validation, Pino structured logging, Vitest + Supertest testing. Single Docker Compose service (MySQL only); the API runs as a plain Node process.

---

## Technology Stack

| Concern | Technology / Library | Version |
|---|---|---|
| Runtime | Node.js | >=20 |
| Language | TypeScript | 5.6.3 |
| HTTP Framework | Express | 4.21.1 |
| Database | MySQL | 8.0 (via Docker Compose) |
| ORM | Prisma Client | 5.22.0 |
| Auth | jsonwebtoken (JWT HS256) + bcrypt | 9.0.2 / 5.1.1 |
| Validation | Zod | 3.23.8 |
| Logging | Pino + pino-http | 9.5.0 / 10.3.0 |
| ID generation | uuid (v4) | 11.0.3 |
| Testing | Vitest + Supertest | 2.1.4 / 7.0.0 |
| Dev runner | tsx (watch mode) | 4.19.2 |
| Linting | ESLint + typescript-eslint | 8.57.1 / 8.13.0 |
| Formatting | Prettier | 3.3.3 |
| Module system | ESM (`"type": "module"`) | — |

---

## System Modules

### MOD-001 — Infrastructure / Application Bootstrap

- **Files:**
  - `src/server.ts`
  - `src/app.ts`
  - `src/routes/index.ts`
  - `src/config/env.ts`
  - `src/config/database.ts`
  - `docker-compose.yml`
- **Scope:** Small
- **Description:**
  `src/server.ts` is the process entry point: it calls `buildApp({ prisma })`, starts the HTTP listener, and registers `SIGINT`/`SIGTERM` shutdown handlers that close the HTTP server and disconnect Prisma.
  `src/app.ts` wires all Express middleware and routers via `buildControllers(prisma)` and `buildApiRouter(controllers)`. Dependency injection is manual — each repository, service, and controller is instantiated in `buildControllers` and passed down by constructor.
  `src/config/env.ts` parses environment variables with a Zod schema at startup; the process exits immediately if any required variable is missing or invalid.
  `src/config/database.ts` exports a singleton `PrismaClient` instance (`prisma`) used by the API process. Logging is set to `['warn', 'error']` in production and `['warn', 'error']` in development.
  `src/routes/index.ts` aggregates all domain routers under `/api/v1` and exports the `Controllers` type.
- **Dependencies:** MOD-005 (Shared/Infrastructure), all domain modules for router aggregation.

---

### MOD-002 — Auth Module

- **Files:**
  - `src/modules/auth/auth.controller.ts`
  - `src/modules/auth/auth.service.ts`
  - `src/modules/auth/auth.routes.ts`
  - `src/modules/auth/auth.schemas.ts`
- **Scope:** Small
- **Description:**
  Handles user registration, login, and identity retrieval (`GET /api/v1/auth/me`).
  `AuthService.login` verifies the bcrypt password hash, then issues a JWT with `{ sub, email, role }` claims signed with `JWT_SECRET` and expiry from `JWT_EXPIRES_IN`.
  `AuthService.register` delegates to `UserService.createUser`.
  Zod schemas: `registerSchema` (email, password 8–72 chars, name, role enum) and `loginSchema`.
  No route in this module requires the `authenticate` middleware except `GET /auth/me`.
- **Dependencies:** MOD-003 (Users — shares `UserRepository` and `UserService`), MOD-005 (Shared errors, env config).

---

### MOD-003 — Users Module

- **Files:**
  - `src/modules/users/user.controller.ts`
  - `src/modules/users/user.service.ts`
  - `src/modules/users/user.repository.ts`
  - `src/modules/users/user.routes.ts`
  - `src/modules/users/user.schemas.ts`
- **Scope:** Small
- **Description:**
  Manages `User` entities (model: `users` table). Only one public endpoint: `GET /api/v1/users/:id`, which requires `authenticate` + `requireRole('ADMIN')`.
  `UserService.createUser` hashes the password with bcrypt (10 rounds) before persisting.
  `UserService.toPublic(user)` strips `passwordHash` from responses — this is the project's pattern for projection.
  `UserRepository` wraps Prisma calls: `findById`, `findByEmail`, `create`.
  Zod schemas: `createUserSchema`, `userIdParamSchema`.
  `PublicUser` type omits `passwordHash`.
- **Dependencies:** MOD-005 (Shared errors, Prisma client type).

---

### MOD-004 — Customers Module

- **Files:**
  - `src/modules/customers/customer.controller.ts`
  - `src/modules/customers/customer.service.ts`
  - `src/modules/customers/customer.repository.ts`
  - `src/modules/customers/customer.routes.ts`
  - `src/modules/customers/customer.schemas.ts`
- **Scope:** Medium
- **Description:**
  Full CRUD for `Customer` entities (model: `customers` table, JSON `address` field).
  `CustomerService` enforces email uniqueness, delegates list/pagination to repository.
  `CustomerRepository` supports full-text-style OR search across `name`, `email`, and `document` fields. Uses `$transaction([findMany, count])` for paginated list.
  All routes require `authenticate` middleware; no role restriction beyond authentication.
  Zod schemas for create, update (partial), list query with pagination and search.
  `paginated()` helper from `src/shared/http/response.ts` is used for responses.
- **Dependencies:** MOD-005 (Shared errors, response helpers, Prisma).

---

### MOD-005 — Shared / Infrastructure

- **Files:**
  - `src/shared/errors/app-error.ts`
  - `src/shared/errors/http-errors.ts`
  - `src/shared/errors/index.ts`
  - `src/shared/http/response.ts`
  - `src/shared/logger/index.ts`
  - `src/middlewares/auth.middleware.ts`
  - `src/middlewares/error.middleware.ts`
  - `src/middlewares/validate.middleware.ts`
  - `src/middlewares/request-logger.middleware.ts`
- **Scope:** Medium
- **Description:**
  **Error hierarchy:** `AppError` (base class with `statusCode`, `errorCode`, `details`) extended by `BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, `InsufficientStockError`.
  **Error middleware** (`errorMiddleware`): handles `AppError` (returns JSON with `{ error: { code, message, details } }`), `ZodError` (returns `VALIDATION_ERROR` with field-level details), `Prisma.PrismaClientKnownRequestError` for P2002 (unique constraint → 409) and P2025 (not found → 404), and all other errors (logs via Pino and returns 500).
  **Auth middleware:** `authenticate` verifies the `Authorization: Bearer <token>` header using `jwt.verify` against `env.JWT_SECRET`, attaches `req.user: AuthUser`. `requireRole(...roles)` checks `req.user.role` and throws `ForbiddenError` if not in the allowed set.
  **Validate middleware:** `validate({ body?, query?, params? })` parses request parts with Zod schemas, converts `ZodError` to `ValidationError` before passing to next middleware.
  **Request logger:** generates/propagates `X-Request-Id` (UUID v4), logs `http_request` events on `res.finish` via Pino with method, path, statusCode, durationMs, and userId.
  **Logger:** Pino instance with `level` from `env.LOG_LEVEL`, structured JSON in production, `pino-pretty` in development. Redacts `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`.
  **Response helpers:** `paginated<T>()` and `buildPagination()` in `src/shared/http/response.ts`.
- **Dependencies:** `src/config/env.ts` (logger reads LOG_LEVEL).

---

### MOD-006 — Products Module

- **Files:**
  - `src/modules/products/product.controller.ts`
  - `src/modules/products/product.service.ts`
  - `src/modules/products/product.repository.ts`
  - `src/modules/products/product.routes.ts`
  - `src/modules/products/product.schemas.ts`
- **Scope:** Medium
- **Description:**
  Full CRUD for `Product` entities (`products` table). Key fields: `sku` (unique), `priceCents` (integer cents), `stockQuantity`, `active` flag.
  `ProductService` enforces SKU uniqueness, supports update and soft-delete (hard delete in DB).
  `ProductRepository` supports search by `name` or `sku`, filter by `active`, paginated list using `$transaction([findMany, count])`.
  `findManyByIds(ids)` is used by the Orders module during order creation to validate product existence.
  Stock quantity is managed transactionally by `OrderService.changeStatus` (debit on PENDING→PAID, replenish on PAID/PROCESSING→CANCELLED); `ProductRepository` itself has no stock logic.
  All routes require `authenticate`.
- **Dependencies:** MOD-005 (Shared errors, Prisma). Referenced by MOD-007.

---

### MOD-007 — Orders Module

- **Files:**
  - `src/modules/orders/order.controller.ts`
  - `src/modules/orders/order.service.ts`
  - `src/modules/orders/order.repository.ts`
  - `src/modules/orders/order.routes.ts`
  - `src/modules/orders/order.schemas.ts`
  - `src/modules/orders/order.status.ts`
- **Scope:** Large
- **Description:**
  The core business domain. Manages the full lifecycle of `Order` entities, their `OrderItem` children, `OrderStatusHistory` audit records, and sequential `orderNumber` generation.

  **State machine** (`order.status.ts`): the `transitions` map encodes the allowed status graph:
  - `PENDING` → `PAID`, `CANCELLED`
  - `PAID` → `PROCESSING`, `CANCELLED`
  - `PROCESSING` → `SHIPPED`, `CANCELLED`
  - `SHIPPED` → `DELIVERED`
  - `DELIVERED` → (terminal)
  - `CANCELLED` → (terminal)

  Exported functions: `canTransition(from, to)`, `allowedTransitions(from)`, `isTerminal(status)`, `shouldDebitStock(from, to)` (true only for PENDING→PAID), `shouldReplenishStock(from, to)` (true for PAID/PROCESSING→CANCELLED).

  **`OrderService.changeStatus`** runs inside `prisma.$transaction(async (tx) => { ... })` and:
  1. Fetches order with items under lock.
  2. Validates the transition via `canTransition`.
  3. Conditionally calls `debitStock` (validates stock, then decrements `stockQuantity` per item) or `replenishStock`.
  4. Updates `orders.status`.
  5. Inserts a row into `order_status_history`.
  6. Re-fetches the full order with relations and returns it.

  **`OrderService.create`** also runs in a transaction: creates the order, items, initial history entry, and reserves an order number via `orderNumberSequence` upsert.

  **`OrderRepository`**: `list` with filter/pagination using `$transaction([findMany, count])`, `findByIdWithRelations` (eager loads items→product, history, customer), `findById`, `deleteById`.

  **Delete constraint**: orders can only be deleted while in `PENDING` or `CANCELLED` status (enforced in service).

  All routes require `authenticate`. No additional role restriction on any order endpoint.
- **Dependencies:** MOD-004 (Customer lookup), MOD-006 (Product lookup, stock update), MOD-005 (Shared errors, response helpers, Prisma).

---

### MOD-008 — Webhooks Module (Planned)

- **Files (to be created):**
  - `src/modules/webhooks/webhook.controller.ts`
  - `src/modules/webhooks/webhook.service.ts`
  - `src/modules/webhooks/webhook.repository.ts`
  - `src/modules/webhooks/webhook.routes.ts`
  - `src/modules/webhooks/webhook.schemas.ts`
  - `src/modules/webhooks/webhook.processor.ts` (or `webhook.worker.ts`)
  - `src/worker.ts` (new process entry point)
- **Scope:** Large
- **Description:**
  Outbound webhook notification system for order status change events, targeting B2B customers (Atlas Comercial, MaxDistribuição, Nova Cargo).
  Architecture: Transactional Outbox pattern in MySQL. When `OrderService.changeStatus` commits, a new function `publishWebhookEvent(tx, order, fromStatus, toStatus)` inserts a row into `webhook_outbox` within the same transaction, guaranteeing atomic event capture.
  A separate Node process (`src/worker.ts`) polls `webhook_outbox` every 2 seconds, dispatches HTTP POST requests to registered endpoint URLs, implements exponential backoff retry (5 attempts: 1m/5m/30m/2h/12h), and moves permanently-failed events to `webhook_dead_letter`.
  HMAC-SHA256 signing: each registered webhook endpoint has a unique secret stored in the `webhook_endpoints` table; the worker signs each outgoing payload and sends the signature in the `X-Signature` header.
  CRUD endpoints for webhook endpoint configuration are authenticated (any role). The replay endpoint (`POST /admin/webhooks/dead-letter/:id/replay`) requires `requireRole('ADMIN')`.
  Follows identical module structure to existing modules: controller → service → repository → routes → schemas. Error codes use `WEBHOOK_` prefix.
- **Dependencies:** MOD-007 (integration point: `OrderService.changeStatus`), MOD-005 (AppError, Pino logger, error middleware, auth middleware, validate middleware, Prisma client), MOD-004 (customer_id association).

---

## Cross-Cutting Concerns

### Authentication & Authorization

- **JWT issuance:** `AuthService.signToken` in `src/modules/auth/auth.service.ts` — signs `{ sub, email, role }` with `HS256` using `env.JWT_SECRET`, expiry from `env.JWT_EXPIRES_IN`.
- **JWT verification middleware:** `authenticate` in `src/middlewares/auth.middleware.ts` — reads `Authorization: Bearer <token>`, calls `jwt.verify`, attaches `req.user: AuthUser { id, email, role }`.
- **Role enforcement:** `requireRole(...roles): RequestHandler` in `src/middlewares/auth.middleware.ts` — used by `buildUserRouter` (requires `'ADMIN'`). The webhook DLQ replay endpoint will also require `requireRole('ADMIN')`.
- **Roles defined in Prisma schema:** `enum UserRole { ADMIN, OPERATOR }`.
- **Pattern:** Routes that need auth apply `authenticate` at the router level (e.g., `router.use(authenticate)` in order routes); individual routes additionally call `requireRole` when role restriction is needed.

### Error Handling

- **Base class:** `AppError` (`src/shared/errors/app-error.ts`) — properties: `statusCode`, `errorCode` (string constant like `INVALID_STATUS_TRANSITION`), `details` (optional).
- **Concrete error classes** (`src/shared/errors/http-errors.ts`):
  - `BadRequestError` (400, `BAD_REQUEST`)
  - `ValidationError` (400, `VALIDATION_ERROR`)
  - `UnauthorizedError` (401, `UNAUTHORIZED`)
  - `ForbiddenError` (403, `FORBIDDEN`)
  - `NotFoundError` (404, `NOT_FOUND`)
  - `ConflictError` (409, `CONFLICT`)
  - `UnprocessableEntityError` (422, `UNPROCESSABLE_ENTITY`)
  - `InvalidStatusTransitionError` (409, `INVALID_STATUS_TRANSITION`)
  - `InsufficientStockError` (422, `INSUFFICIENT_STOCK`)
- **Central middleware:** `errorMiddleware` in `src/middlewares/error.middleware.ts` — catches and formats `AppError`, `ZodError`, and Prisma errors; logs unhandled errors via `logger.error`.
- **Convention for new modules:** Webhook module will add error codes with `WEBHOOK_` prefix (e.g., `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) as sub-classes of existing base classes.

### Logging

- **Logger setup:** `src/shared/logger/index.ts` — `createLogger()` returns a Pino instance. Base fields: `{ service: 'order-management-api', env }`. ISO timestamps. Pretty-print in development via `pino-pretty`.
- **Redaction:** `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` are censored as `[REDACTED]`.
- **HTTP request logging:** `requestLogger` middleware in `src/middlewares/request-logger.middleware.ts` — logs `http_request` on response finish with `requestId`, `method`, `path`, `statusCode`, `durationMs`, `userId`.
- **Application log events:** `server_started`, `shutdown_initiated`, `http_server_closed`, `bootstrap_failed` in `src/server.ts`.
- **Worker:** The planned `src/worker.ts` and `src/modules/webhooks/webhook.processor.ts` will reuse the same `logger` instance (confirmed in TRANSCRICAO.md [09:29]).

### Validation

- **Library:** Zod 3.23.8, all schemas in `*.schemas.ts` files per module.
- **Middleware:** `validate({ body?, query?, params? })` in `src/middlewares/validate.middleware.ts` — parses request parts with Zod, wraps `ZodError` in `ValidationError` before passing to next.
- **Environment validation:** `src/config/env.ts` uses a dedicated Zod schema (`envSchema`) at startup; exits with details on failure.
- **Schema examples:**
  - `orderIdParamSchema` — UUID param validation
  - `updateOrderStatusSchema` — `toStatus: z.nativeEnum(OrderStatus)`, optional `reason`
  - `createOrderSchema` — nested array of items, `discountCents` nonnegative
  - `listOrdersQuerySchema` — coerced pagination params, optional enum filter, optional date range
- **Webhook schemas (planned):** URL validation with `https` enforcement, `event_types` array filter (subset of `OrderStatus`), secret validation.

### Data Access (Prisma Patterns)

- **Singleton client:** `prisma` exported from `src/config/database.ts` — shared across all repositories in the API process.
- **Worker client:** A separate `PrismaClient` instance will be created in `src/worker.ts` for the worker process (same `DATABASE_URL`, different process — per [09:30] Bruno in TRANSCRICAO.md).
- **Repository pattern:** Each module has a `*Repository` class receiving `PrismaClient` via constructor. Direct Prisma calls are isolated inside repositories; services receive repository instances.
- **Transactions:** `prisma.$transaction(async (tx) => { ... })` used in `OrderService.create` and `OrderService.changeStatus` to ensure atomicity across multiple table writes. `OrderRepository.list` and other repositories use `$transaction([promise1, promise2])` (batch reads) for consistent pagination counts.
- **Type from Prisma:** `Prisma.TransactionClient` aliased as `TxClient` in `order.service.ts` — the `tx` parameter type for functions that operate inside a transaction. The webhook outbox insertion function `publishWebhookEvent(tx, ...)` will receive this type.
- **UUID primary keys:** All entity IDs use `@id @default(uuid()) @db.Char(36)` — consistent across all models. The planned webhook tables will follow the same convention (confirmed [09:51] Larissa).
- **Indexed fields:** `orders` table indexes on `customerId`, `status`, `createdAt`, `createdById`. Planned `webhook_outbox` will need indexes on `status` and `created_at` for the worker polling query.

### Configuration

- **Source:** `src/config/env.ts` — single point of truth for all environment variables.
- **Variables:** `NODE_ENV`, `PORT` (default 3000), `LOG_LEVEL` (default `info`), `DATABASE_URL` (required), `JWT_SECRET` (min 16 chars, required), `JWT_EXPIRES_IN` (default `8h`).
- **Shadow DB:** `prisma/schema.prisma` references `SHADOW_DATABASE_URL` for Prisma migrations.
- **Scripts:** `npm run dev` (tsx watch), `npm run build` (tsc), `npm start` (dist/server.js), `npm test` (vitest run). The planned worker will add `npm run worker` pointing to `src/worker.ts`.

---

## Key Architectural Patterns

### State Machine (OrderStatus Transitions)

- Encoded in `src/modules/orders/order.status.ts` as a plain `transitions` map (`Record<OrderStatus, ReadonlyArray<OrderStatus>>`).
- Functions: `canTransition(from, to)`, `allowedTransitions(from)`, `isTerminal(status)`.
- Stock side-effects are co-located: `shouldDebitStock` (PENDING→PAID) and `shouldReplenishStock` (PAID/PROCESSING→CANCELLED).
- The transition check and stock mutations happen inside the same `prisma.$transaction` in `OrderService.changeStatus` — this is the exact integration point for the webhook outbox write.

### Repository Pattern

- Each domain module has a `*Repository` class as the sole data access layer.
- Repositories accept `PrismaClient` in the constructor, are never aware of HTTP or business rules.
- Services receive repository instances; controllers receive service instances.
- Cross-module access to data goes through the other module's service or repository (e.g., `OrderService` calls `tx.customer.findUnique` directly inside the transaction because it needs the same tx client, bypassing `CustomerRepository`).

### Controller → Service → Repository Layering

- `Controller`: handles HTTP concerns — extracts `req.body`/`req.params`/`req.user`, delegates to service, sends response or calls `next(err)`.
- `Service`: contains all business logic, throws typed `AppError` subclasses for domain violations.
- `Repository`: contains all Prisma calls, returns Prisma model types or `null`.
- This 3-layer pattern is applied uniformly across all modules (Auth, Users, Customers, Products, Orders) and will be replicated for Webhooks.

### Error Code Conventions

- Format: `SCREAMING_SNAKE_CASE` string constants.
- Domain-specific codes are prefixed by domain: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`, `EMAIL_ALREADY_USED`, `SKU_ALREADY_USED`, `INACTIVE_PRODUCT`, `INVALID_ORDER_STATE_FOR_DELETE`.
- HTTP-generic codes: `BAD_REQUEST`, `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `UNPROCESSABLE_ENTITY`, `INTERNAL_SERVER_ERROR`.
- Planned webhook codes: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, etc. (per [09:28–09:29] Bruno).

### Transaction Handling

- `prisma.$transaction(async (tx) => { ... })` used for multi-step atomic writes.
- Functions that must participate in an existing transaction receive the `tx: Prisma.TransactionClient` as a parameter (see `debitStock(tx, items)`, `replenishStock(tx, items)`, `reserveOrderNumber(tx)` in `order.service.ts`).
- This pattern makes the planned `publishWebhookEvent(tx, order, fromStatus, toStatus)` function natural — it receives the tx from `changeStatus` and inserts into `webhook_outbox` within the same commit.

### Dependency Injection (Manual / Constructor-Based)

- `buildControllers(prisma)` in `src/app.ts` is the composition root. All dependencies are wired explicitly with `new` calls.
- No IoC container. Testability relies on passing mocks/fakes via constructors in test setup.

---

## Integration Points for Webhook Feature

### 1. Status Change Event Emission

- **File:** `src/modules/orders/order.service.ts`
- **Method:** `OrderService.changeStatus` (lines 126–178)
- **Integration:** After `tx.orderStatusHistory.create(...)` and before the final `tx.order.findUnique` re-fetch, call `await publishWebhookEvent(tx, order, from, to)`. This function is the boundary between the Orders module and the Webhooks module. It runs inside `this.prisma.$transaction(async (tx) => { ... })`, so the outbox insert is atomic with the status update.
- **Function signature proposed in TRANSCRICAO.md [09:41]:** `publishWebhookEvent(tx: Prisma.TransactionClient, order: Order & { items: OrderItem[] }, fromStatus: OrderStatus, toStatus: OrderStatus): Promise<void>`
- **Payload snapshot:** The function must render the full event payload at insertion time (per [09:52] Larissa: "snapshot na inserção"), so the `webhook_outbox.payload` column stores the pre-rendered JSON rather than just the `order_id`.

### 2. Outbox Table Integration (Prisma Schema)

- **File:** `prisma/schema.prisma`
- **Change needed:** Add two new Prisma models — `WebhookOutbox` and `WebhookDeadLetter` — plus a `WebhookEndpoint` model for customer endpoint configuration.
- **`WebhookOutbox` fields (from TRANSCRICAO.md):** `id` (UUID), `webhookEndpointId`, `eventId` (UUID, unique — the `X-Event-Id`), `eventType` (e.g., `order.status_changed`), `payload` (JSON, rendered at insert time), `status` (`PENDING` | `PROCESSING` | `FAILED` | `DELIVERED`), `retryCount`, `nextRetryAt`, `createdAt`. Indexes on `status` and `createdAt` for worker polling.
- **`WebhookDeadLetter` fields:** `id` (UUID), `webhookEndpointId`, `originalOutboxId`, `payload`, `failureReason`, `failedAt`.
- **`WebhookEndpoint` fields (from [09:21] Sofia):** `id` (UUID), `customerId` (FK to Customer), `url` (HTTPS only), `secret` (per-endpoint, rotatable), `active` (Boolean), `eventTypes` (JSON array of `OrderStatus` values), `createdAt`, `updatedAt`.

### 3. Worker Process Entry Point

- **File to create:** `src/worker.ts`
- **Pattern:** Mirrors `src/server.ts` in structure — imports a new `createPrismaClient()` call (separate instance from the API's `prisma` singleton), instantiates `WebhookProcessor` or equivalent class, starts the polling loop, handles `SIGINT`/`SIGTERM` for graceful shutdown.
- **npm script to add:** `"worker": "tsx watch --env-file=.env src/worker.ts"` (and `"worker:start": "node --env-file=.env dist/worker.js"` for production).
- **Polling logic** (`src/modules/webhooks/webhook.processor.ts`): every 2 seconds, queries `webhook_outbox` for `status = PENDING` ordered by `createdAt ASC`, processes each batch, dispatches HTTP POST with 10s timeout, handles retry/DLQ logic.

### 4. Shared Utilities to Reuse

| Utility | File | Usage in Webhooks |
|---|---|---|
| `AppError` + subclasses | `src/shared/errors/app-error.ts`, `http-errors.ts` | New `WEBHOOK_*` error subclasses extend existing hierarchy |
| `errorMiddleware` | `src/middlewares/error.middleware.ts` | No change needed — already handles `AppError`, `ZodError`, Prisma errors. Webhook errors propagate automatically |
| `authenticate` | `src/middlewares/auth.middleware.ts` | Applied to webhook CRUD routes |
| `requireRole('ADMIN')` | `src/middlewares/auth.middleware.ts` | Applied to `POST /admin/webhooks/dead-letter/:id/replay` |
| `validate(schemas)` | `src/middlewares/validate.middleware.ts` | Applied to all webhook routes with Zod schemas |
| `logger` | `src/shared/logger/index.ts` | Worker and processor log via same Pino instance (new import in worker process) |
| `env` | `src/config/env.ts` | Worker reads `DATABASE_URL`, `LOG_LEVEL`, `NODE_ENV` |
| `createPrismaClient()` | `src/config/database.ts` | Worker calls `createPrismaClient()` to get its own PrismaClient instance |
| `paginated()` | `src/shared/http/response.ts` | Webhook delivery history list response |
| `uuid` (v4) | External package `uuid` | Generate `event_id` (UUID) when inserting into `webhook_outbox` |

---

## Guidelines for Phase 2 (ADR Identification)

### Highest Priority Modules for ADR Analysis

1. **MOD-007 (Orders)** — The critical integration point. The `changeStatus` transaction is where the webhook outbox write will be injected. Any architectural decision that affects atomicity guarantees flows through here.
2. **MOD-008 (Webhooks — Planned)** — The feature itself. Every architectural decision made in TRANSCRICAO.md concerns this module's design.
3. **MOD-005 (Shared/Infrastructure)** — Decisions about error patterns, logging, and middleware reuse directly affect how the webhook module is constrained.

### Key Decisions Already Visible in the Code

- **UUID for all PKs:** `@id @default(uuid())` on every model. The webhook outbox and endpoint tables must follow this (confirmed [09:51]).
- **Cents for monetary values:** `priceCents`, `subtotalCents`, `discountCents`, `totalCents` — all integer cents. Consistent with no floating-point money.
- **Manual DI / no IoC container:** `buildControllers(prisma)` in `src/app.ts`. The webhook module's controller/service/repository will be added to this function.
- **ESM module system:** `"type": "module"` in `package.json`. All imports use `.js` extension. New files must follow the same convention.
- **Single-process API, separate worker process:** Explicit from `src/server.ts` pattern and [09:11] Larissa.

### Decisions from TRANSCRICAO.md that Need ADRs

1. **[09:06–09:08] Diego, Larissa** — **Transactional Outbox in MySQL over Redis Streams or synchronous HTTP.** Reason: team size, existing infrastructure, atomicity guarantee. Rejected alternatives: synchronous webhook call in `changeStatus`, Redis Streams/Cluster.

2. **[09:09–09:10] Diego, Larissa** — **Worker polling at 2-second intervals over MySQL NOTIFY-style triggers.** Reason: MySQL has no native external listener; polling satisfies the <10s latency SLA. Rejected: database triggers with external notification.

3. **[09:15–09:17] Diego, Larissa** — **Retry policy: 5 attempts with exponential backoff (1m/5m/30m/2h/12h) and DLQ in a separate table.** Reason: covers up to ~15-hour client outage window; DLQ table keeps `webhook_outbox` clean. Rejected: 3 attempts (too aggressive), retry indefinitely (events can be stranded).

4. **[09:20–09:22] Sofia, Larissa** — **HMAC-SHA256 authentication with per-endpoint unique secrets and 24-hour rotation grace period.** Reason: prevents payload tampering; per-endpoint isolation limits blast radius of a secret leak. Rejected: single global platform secret.

5. **[09:24–09:26] Diego, Larissa** — **At-least-once delivery with client-side deduplication via `X-Event-Id`.** Reason: exactly-once requires two-sided coordination; at-least-once with UUID event ID is the industry standard (Stripe, GitHub). Trade-off: client responsibility for deduplication.

6. **[09:27–09:30] Bruno, Diego, Larissa** — **Webhook module follows existing Controller → Service → Repository + Zod schemas pattern; reuses AppError, Pino, error middleware; WEBHOOK_ error code prefix.** Reason: consistency, zero new infrastructure. Rejected: introducing new frameworks or IoC containers.

7. **[09:51–09:52] Larissa, Diego, Bruno** — **Outbox stores pre-rendered payload snapshot at insert time.** Reason: event reflects the exact order state at the moment the status changed; late rendering could produce inconsistent snapshots if order data changes before dispatch.

8. **[09:11, 09:29–09:30] Diego, Bruno** — **Worker runs as a separate Node.js process with its own PrismaClient instance.** Reason: API restarts must not interrupt the worker; Node.js is single-threaded and process isolation provides fault boundary.
