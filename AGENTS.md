## Purpose
This repository is a NestJS backend. Prefer predictable, explicit, maintainable solutions over clever abstractions.

## Core principles
- Keep modules, services, and functions focused on one responsibility.
- Prefer explicit types, explicit validation, and explicit failure handling.
- Do not silently ignore errors, missing data, or failed async work.
- Design every async/background workflow for retries and recovery.

## Error handling
- Do not use `try/catch` unless you are:
  - adding meaningful context
  - translating an external/infrastructure error
- Never swallow errors.
- Let the global exception layer handle application errors by default.
- In BullMQ processors, do not wrap the whole processor in `try/catch`; let BullMQ manage retries and failed state.
- Log failures with relevant entity IDs.

## Async jobs
- Jobs must be idempotent when duplicate execution is possible.
- Always set deterministic `jobId` where idempotency matters.
- Every async workflow must define:
  - retry strategy
  - failure recovery strategy
- Fire-and-forget promises must have an explicit `.catch(...)` handler.
- Worker failures must be observable, e.g. via worker event handlers and structured logs.

## Validation
- Validate all incoming data: params, query, body, events, and job payloads.
- Validate format, bounds, lengths, and uniqueness where applicable.
- Normalize input when needed (e.g. trimming strings).
- Do not pass raw external input deeper into the system without validation.

## Logging
- Error logs must include useful identifiers and context.
- Avoid vague logs such as “operation failed” without entity IDs.
- Prefer structured logs over ad-hoc string logs.

## Type safety
- Exhaustively handle unions/enums in `switch` statements.
- Prefer object parameters over multiple positional parameters of the same type.
- Use precise DTOs/types instead of loose objects.
- Prefer JSDoc for public or non-obvious code; avoid redundant inline comments.

## Architecture
- Keep domain boundaries clear.
- Do not couple domains by exporting internal domain services directly unless there is an intentional shared boundary.
- Separate domain logic from infrastructure concerns.
- Each domain should have its own module.
- Prefer shared/domain library abstractions only when the logic is truly cross-domain.

## Code structure
- Split large files, classes, and functions before they become difficult to reason about.
- Prefer orchestration methods that call smaller private methods over one large implementation block.
- Avoid dead abstractions and speculative generalization.

## Performance and concurrency
- Do not use unbounded `Promise.all(...)` for heavy I/O, DB writes, or external API calls.
- Prefer bulk DB operations where supported.
- Use concurrency limiting for parallel async work.

## Jobs and scaling
- Keep scalable worker processing separate from singleton/scheduled cron behavior.
- Do not assume cron-like logic is safe to run on every instance.

## API and GraphQL
- GraphQL types should expose domain relationships intentionally, not just database foreign keys.
- Prefer nested relation fields over exposing raw related entity IDs.
- Resolve async GraphQL relations through `@ResolveField` with DataLoader, not direct per-field service/database calls.
- Keep API shape aligned with actual domain relationships.

## Numeric correctness
- Do not use JS floating-point arithmetic for money or precision-sensitive calculations.
- Use a decimal/arbitrary-precision library where correctness matters.

## Collections and loops
- Do not silently skip mandatory entities in loops.
- If required data is missing, throw or log explicitly with identifiers.

## Repository structure
- `/domain` contains business/domain modules.
- `/infrastructure` contains framework and integration concerns.

## Before finishing work
- Run lint.
- Run tests relevant to the change.
- Run typecheck.
- Update validation, schemas/contracts, and tests together when changing inputs or outputs.

## Rules for updating this file
Only add rules that are:
- repo-wide
- stable
- costly to ignore
- not better enforced by tooling
