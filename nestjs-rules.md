# NestJS Backend Rules

## Error Handling

- Do not abuse `try/catch` — let the global exception handler do its job. Only use `try/catch` when adding context or translating external errors.
- Never silently swallow errors — no empty `catch` blocks, always log or rethrow.
- Do not wrap BullMQ processors in `try/catch` — let BullMQ handle retries and failure states.
- Handle worker errors explicitly using `@OnWorkerEvent('failed')` with structured logging (jobId, entityId, error message).
- Always attach `.catch()` to fire-and-forget promises with logging inside.

## Async Jobs & Idempotency

- Always set a deterministic `jobId` (e.g. `send-email:${userId}`) to make jobs idempotent.
- Every async operation must define a retry strategy and a recovery strategy (cron retry, manual reprocessing, or dead-letter queue).

## Logging

- Always log entity IDs (userId, orderId, etc.) on errors — never log bare messages without context.
- Do not use `logger.debug` for important events — use `logger.log`, `logger.warn`, or `logger.error` based on severity. `debug` is often disabled in production.

## Validation

- Validate ALL incoming data: params, query, and body.
- Trim strings, enforce min/max lengths, validate arrays for both length and uniqueness.
- Use Zod (preferred) or class-validator for validation, depending on what the project already uses.

## Type Safety

- Use exhaustive `switch` with `satisfies never` in the default case to catch unhandled enum/union values at compile time.

## Architecture

- Do not export services directly from domain modules — this creates tight coupling. Use shared "domain lib" modules for cross-domain logic.
- Separate domain code (`/domain/user`, `/domain/patient`) from infrastructure (`/infrastructure/auth`, `/infrastructure/bullmq`).
- Each domain gets its own module — do not bundle multiple domains into one module.

## Code Structure

- Keep files under 1000 lines — split large logic into multiple services.
- Keep functions under 100 lines — decompose into named steps (validate, calculate, persist).

## Performance & Concurrency

- Do not use `Promise.all` on database operations — use bulk/batch methods like `insertMany` instead.
- Use `p-limit` for concurrency control when running multiple async operations in parallel.

## Jobs & Scaling

- Separate workers (horizontally scalable) from cron jobs (single instance).

## API Design

- Use an object parameter instead of multiple positional parameters of the same type.

## Numbers

- Never use native JS float arithmetic for money or precision-sensitive calculations — use `decimal.js`.
- Be aware of division rounding — always choose an explicit rounding strategy (round, truncate, or full precision). Never silently pass unrounded division results to APIs, databases, or user-facing output.

## Documentation

- Prefer JSDoc (`/** */`) over inline comments (`//`) for functions, classes, and complex variables.
- Keep JSDoc short — one line for simple things, max 2-3 lines for complex logic. No one wants to read a wall of documentation.
- Every `Map` must have a single-line JSDoc describing key -> value meaning, especially `Map<string, string>`.

## Async Functions

- In `async` functions, use `return await ...` instead of returning a raw promise — keeps error handling and stack traces consistent.

## GraphQL

- Never expose foreign key IDs (userId, patientId) in GraphQL types — expose related entities as nested `@ResolveField` instead.
- Always resolve relations via `@ResolveField` + DataLoader — never call services/repositories directly inside a resolver.
- Keep GraphQL schema aligned with DB relation structure — if a link/join table exists, represent it explicitly in the schema (e.g. `User.memberships: [Membership]` not `User.projects: [Project]`).

## Loops & Mandatory Entities

- Do not silently `continue` when a mandatory entity is missing inside a loop — either throw an error or log with entity IDs.
