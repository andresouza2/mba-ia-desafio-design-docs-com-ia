# CLAUDE.md — Order Management System (OMS) — Webhook Design Docs Challenge

## Project Overview

This is a **documentation-only challenge**. The goal is to produce a complete design docs package for a **Webhook Notification System** feature on top of an existing Order Management System (OMS).

**Do NOT modify any code** in `src/`, `prisma/`, `tests/`, or configuration files. All work is documentary.

## Existing Application Stack

- **Runtime**: Node.js ≥ 20, TypeScript (ESM modules)
- **Framework**: Express 4
- **Database**: MySQL via Prisma ORM
- **Auth**: JWT (`jsonwebtoken`) + bcrypt, role-based (`ADMIN` | `OPERATOR`)
- **Validation**: Zod schemas
- **Logging**: Pino + pino-http
- **Testing**: Vitest + Supertest

## Source Code Structure

```
src/
  app.ts                  # Express app setup
  server.ts               # HTTP server entry
  config/                 # Config (DB, JWT, etc.)
  middlewares/            # requireRole, error handler, etc.
  routes/                 # Root router aggregation
  shared/                 # Error classes, shared utilities
  modules/
    auth/                 # JWT login flow
    users/                # User CRUD (ADMIN only)
    customers/            # Customer CRUD
    products/             # Product CRUD + stock
    orders/               # Order lifecycle, state machine, audit
      order.service.ts    # changeStatus(), stock transactions
      order.status.ts     # OrderStatus enum / state machine
      order.repository.ts
      order.controller.ts
      order.routes.ts
      order.schemas.ts
```

## Key Patterns in the Codebase

- **State machine**: `src/modules/orders/order.status.ts` controls valid `OrderStatus` transitions
- **changeStatus**: `src/modules/orders/order.service.ts` — transactional status update + stock control + audit
- **Error classes**: `src/shared/` — structured error codes, reused across modules
- **requireRole middleware**: `src/middlewares/` — RBAC guard
- **Centralized error middleware**: catches all thrown errors and formats response
- **Pino logger**: structured JSON logging via `pino-http`

## Documentation to Produce

Files go in `docs/` (already created, mostly empty):

| File | Status | Description |
|---|---|---|
| `docs/PRD.md` | empty | Product Requirements Document |
| `docs/RFC.md` | empty | Request for Comments (architecture proposal) |
| `docs/FDD.md` | empty | Feature Design Document (implementation spec) |
| `docs/TRACKER.md` | empty | Traceability table linking items to transcription/code |
| `docs/adrs/ADR-NNN-*.md` | empty dir | 5–8 Architecture Decision Records |

## Source Materials

- **`TRANSCRICAO.md`** — Full transcript of the design meeting (~55 min). Primary source for all decisions, requirements, and constraints. Every doc item must trace back here or to the code.
- **`README.md`** — Challenge spec (do not delete; replace content with process documentation at the end)

## The 6 Core Decisions (must be covered in ADRs — at least 5 of 6)

1. Outbox pattern in MySQL (no external message broker)
2. Retry policy with exponential backoff + Dead Letter Queue (DLQ)
3. HMAC-SHA256 authentication with per-endpoint secret
4. At-least-once delivery guarantee via `X-Event-Id` header
5. Separate worker process using polling (not event-driven)
6. Reuse of existing project patterns (error classes, logger, Prisma, etc.)

## Recommended Production Order

1. ADRs → RFC → FDD → PRD → TRACKER → README (process docs)

## Critical Rules

- **No invented requirements**: every item must trace to `TRANSCRICAO.md` (timestamp + speaker) or `src/` (file path)
- **No code modifications**: purely documentary deliverable
- **Tracker coverage**: ≥80% of document items must have a tracker row
- **FDD "Integração com sistema existente"**: must name ≥4 real file paths from the codebase
- **ADR format**: MADR — Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- **Error codes**: FDD error matrix must use `WEBHOOK_*` prefix pattern
- **RFC scope**: architecture-level only (2–4 pages); implementation detail stays in FDD
