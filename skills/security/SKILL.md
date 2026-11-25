---
name: security
description: Global standards for Security, Clean Code, Testing, Error Handling, Logging, Migrations, and API Stability. Scalability and performance live in the `performance` skill; structure/layering in the `architecture` skill.
---

> Scope: this skill covers trust boundaries plus cross-cutting delivery standards (testing, error handling, logging, migrations, API stability). Runtime cost (N+1, indexes, caching, queues, pagination) is the `performance` skill. Code structure/layering/SRP is the `architecture` skill. Avoid duplicating their guidance here.

## 1. Security
- Secrets: Zero tolerance for hardcoded API keys, tokens, or passwords. Use environment variables.
- Inputs: All user inputs must be sanitized and validated using a schema validator (Zod, Joi, class-validator).
- Database: Use parameterized queries to prevent SQL injection. Never concatenate user input into queries.
- Auth: Implement proper authentication and authorization checks on every protected endpoint.
- Headers: Set security headers (CORS, CSP, X-Frame-Options) appropriately.
- Dependencies: No known vulnerable dependencies. Run security audits regularly.

## 2. Code Quality
- Clean Code: Adhere strictly to SOLID and DRY principles.
- Readability: Functions should be concise (max 25 lines) and self-documenting.
- Naming: Use descriptive, consistent naming. Variables reveal intent.
- Standards: Remove dead code, unused imports, and console logs before completion.
- Types: Use strict typing. Avoid `any` type in TypeScript.

## 3. Testing
- Coverage: Minimum 80% coverage for business logic and critical paths.
- Unit Tests: All public functions must have unit tests.
- Integration: API endpoints must have integration tests.
- Mocking: Mock external dependencies (databases, APIs) in unit tests.
- Edge Cases: Test boundary conditions, empty states, and error scenarios.

## 4. Error Handling
- Consistency: Use a consistent error response format: `{ error: { code, message, details? } }`.
- Boundaries: Catch errors at service boundaries. Don't let raw errors leak to clients.
- Logging: Log all errors with context (request ID, user ID, stack trace).
- No Silent Failures: Never swallow errors without logging.
- Graceful Degradation: Handle third-party service failures gracefully.

## 5. Logging & Observability
- Structured Logs: Use JSON format with consistent fields (timestamp, level, message, context).
- Log Levels: ERROR (failures), WARN (potential issues), INFO (business events), DEBUG (development).
- Correlation: Include request/correlation IDs for tracing across services.
- No Sensitive Data: Never log passwords, tokens, or PII.
- Metrics: Track key business and performance metrics.

## 6. Database Migrations
- Versioned: Every schema change requires a migration file.
- Reversible: All migrations must have rollback (down) logic.
- Immutable: Never modify migrations that have been released/deployed.
- Tested: Test rollback locally before deploying.
- Atomic: Each migration should be a single, atomic change.
- No Data Loss: Destructive changes require data backup or migration scripts.

## 7. API Stability
- Backward Compatible: Never break existing clients without versioning.
- Deprecate First: Mark endpoints deprecated before removal (minimum 1 version).
- Versioning: Use URL versioning (`/api/v1/`) from the start.
- Changelog: Document all breaking changes in CHANGELOG.md.
- Contracts: API responses should match documented schemas.
- Graceful Changes: Add new fields, don't rename or remove existing ones.

## 8. Framework Specific Optimization (Laravel Context)
If the project is Laravel, strictly apply these patterns to reduce boilerplate:
* **Request -> DTO -> Service:** Do not pass the raw `Request` object to Services. Extract validated data into a DTO or use `ValidatedRequest` pattern.
* **Guard Clauses via Middleware:** Do not check for user existence or roles inside the controller (e.g., `if (!$user)`). Use Middleware or Policy Gates (`$this->authorize`).
* **Response Classes:** Do not manually build JSON responses (e.g., `response()->json(['status' => true])`). Use a standardized `ApiResponse` class or macros.
* **FormRequest Only:** Never validate input inside the controller method. Always use a dedicated `FormRequest` class.
