# Manual Dependency Injection (No IoC Container)

## Score
Total: 138/150 | Step 0: 75 | Scope+Impact: 23/25 | Cost to Change: 20/25 | Team Knowledge: 20/25

## Evidence
- File: `src/app.ts` — `buildControllers(prisma)` is the composition root: all repositories, services, and controllers are instantiated with `new` and wired by constructor argument; no decorators, no container
- File: `src/config/database.ts` — `createPrismaClient()` factory callable from any process; `prisma` singleton for the API; worker calls `createPrismaClient()` for its own instance
- Transcript: `[09:27] Bruno` — "a gente não precisa inventar nada novo. Segue o mesmo padrão dos outros módulos."
- Transcript: `[09:29] Bruno` — webhook module will be wired into `buildControllers` like all other modules; explicitly rejecting introduction of new frameworks

## Decision Summary
Dependency injection is performed manually at the application composition root (`buildControllers(prisma)` in `src/app.ts`) using plain constructor calls with `new`. No IoC container (tsyringe, InversifyJS, etc.) is used. Each layer receives its dependencies as explicit constructor parameters, making the dependency graph fully visible in one function. This pattern is replicated for every new module, including the planned webhooks module, by adding its controller to `buildControllers`.

## Context
The application was started without an IoC container and the team has no experience with TypeScript DI frameworks. Introducing a container mid-project would require refactoring all existing modules to use decorators (`experimentalDecorators`, `emitDecoratorMetadata`) and would obscure the dependency graph in metadata rather than code. Manual DI is sufficient for the current scale and keeps the codebase framework-agnostic.

## Alternatives Considered
- **tsyringe or InversifyJS (decorator-based IoC):** Requires `experimentalDecorators` and `emitDecoratorMetadata` in `tsconfig.json`; decorators obscure the dependency graph; would require refactoring all existing modules. Explicitly rejected by Bruno at [09:27].
- **NestJS-style framework DI:** Would require rewriting the entire application in NestJS conventions; not viable mid-project.
- **Service locator / global singletons:** Hidden dependencies; hard to test in isolation; violates inversion of control without the benefits of DI.

## Consequences
**Positive:**
- Dependency graph is fully explicit and readable in `buildControllers` — one function to understand the entire wiring
- No build-time decorator compilation required; `tsconfig.json` remains standard
- Constructor injection enables trivial test doubles (pass mock repository to service constructor)
- Adding the webhook module requires only adding its instantiation to `buildControllers`

**Negative / Trade-offs:**
- As the number of modules grows, `buildControllers` becomes a long function; at scale, splitting by domain area may be needed
- No lazy initialization — all dependencies are instantiated at startup regardless of whether they are used in a given request
- Circular dependencies between modules cannot be resolved automatically (not a current issue, but a risk as the graph grows)

## Related Modules
- All modules (MOD-001 is the composition root; MOD-002 through MOD-008 are wired here)
