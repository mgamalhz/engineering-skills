---
name: security
description: Use when writing or reviewing code that handles user input, authentication, authorization, records owned by a user or tenant, file uploads, webhooks, payments, secrets or API responses, and before calling any change done (tests, error handling, logging, migrations, API stability). Covers the common Laravel holes (IDOR, mass assignment, tenancy, rate limiting, XSS, uploads, webhooks). Runtime cost is the `performance` skill; code structure is the `architecture` skill.
---

> Scope: this skill covers trust boundaries plus cross-cutting delivery standards (testing, error handling, logging, migrations, API stability). The error response format and API versioning live here only. Runtime cost (N+1, indexes, caching, queues, pagination) is the `performance` skill. Code structure/layering/SRP is the `architecture` skill. Avoid duplicating their guidance here.

## 1. Security
- Secrets: Zero tolerance for hardcoded API keys, tokens, or passwords. Use environment variables.
- Inputs: All user inputs must be sanitized and validated using a schema validator (Zod, Joi, class-validator).
- Database: Use parameterized queries to prevent SQL injection. Never concatenate user input into queries.
- Auth: Implement proper authentication and authorization checks on every protected endpoint.
- Headers: Set security headers (CORS, CSP, X-Frame-Options) appropriately.
- Dependencies: No known vulnerable dependencies. Run security audits regularly.

## 2. Common Holes (Laravel Context)
These are the bugs that pass every happy-path test. Check each one that the change touches.

- **Record-level authorization (IDOR):** being allowed to call an endpoint is not being allowed to see every record behind it. A user who opens `/orders/15` must own order 15.
  - Authorize the record with a Policy: `$this->authorize('view', $order)` or `Gate::authorize('update', $order)`.
  - Or load it through its owner so other users' records can't be found: `$request->user()->orders()->findOrFail($id)`.
  - Nested routes use `->scopeBindings()` so `/users/1/orders/15` fails when order 15 belongs to someone else.
- **Mass assignment:** every model declares `$fillable`. Never pass `$request->all()` or `$request->input()` to `create`, `update` or `fill`; pass `$request->validated()` or a DTO. Fields like `role`, `is_admin`, `user_id`, `tenant_id`, `price` and `status` are never fillable from user input.
- **Tenancy:** in a multi-tenant app, every query on tenant data is scoped to the current tenant, preferably by a global scope on the model, never by remembering a `where('tenant_id', ...)` in each query. Background jobs set the tenant explicitly; they have no request to read it from.
- **Rate limiting:** login, register, OTP, password reset and any endpoint that sends email or SMS are throttled: `RateLimiter::for('login', ...)` in a service provider and the `throttle:login` middleware on the route. Limit by user or email and by IP.
- **XSS:** Blade `{{ }}` escapes output; `{!! !!}` does not. Use `{!! !!}` only for HTML you generated or sanitized yourself, never for user input. The same applies to `v-html` and `dangerouslySetInnerHTML`.
- **File uploads:**
  - Validate type and size: `'file' => ['required', 'file', 'mimes:pdf,jpg,png', 'max:5120']`. `mimes` checks the file's content, not the client's extension.
  - Store private files on a private disk (`Storage::disk('local')`), not in `public/`, and serve them through a route that authorizes the user, or a temporary signed URL.
  - Never use the client's file name as the storage path; let Laravel generate it (`$file->store(...)`).
- **Webhooks:**
  - Verify the signature before doing anything: compute the HMAC of the raw body with the shared secret and compare with `hash_equals`, never `==`.
  - Process each event once: store the provider's event ID with a unique index and skip duplicates, because providers retry.
  - Return 2xx fast and do the work in a queued job.
- **Response fields:** API Resources list the fields they return explicitly in `toArray`. Never return a model or `$model->toArray()` directly. Passwords, tokens, internal IDs and other users' data are never in a response; `$hidden` on the model is a second line of defense, not the first.

## 3. Code Quality
- Clean Code: Adhere strictly to SOLID and DRY principles.
- Readability: Functions should be concise (max 25 lines) and self-documenting.
- Naming: Use descriptive, consistent naming. Variables reveal intent.
- Standards: Remove dead code, unused imports, and console logs before completion.
- Types: Use strict typing. Avoid `any` type in TypeScript.

## 4. Testing
- Coverage: Minimum 80% coverage for business logic and critical paths.
- Unit Tests: All public functions must have unit tests.
- Integration: API endpoints must have integration tests.
- Mocking: Mock external dependencies (databases, APIs) in unit tests.
- Edge Cases: Test boundary conditions, empty states, and error scenarios.

## 5. Error Handling
- Consistency: Use a consistent error response format: `{ error: { code, message, details? } }`.
- Boundaries: Catch errors at service boundaries. Don't let raw errors leak to clients.
- Logging: Log all errors with context (request ID, user ID, stack trace).
- No Silent Failures: Never swallow errors without logging.
- Graceful Degradation: Handle third-party service failures gracefully.

## 6. Logging & Observability
- Structured Logs: Use JSON format with consistent fields (timestamp, level, message, context).
- Log Levels: ERROR (failures), WARN (potential issues), INFO (business events), DEBUG (development).
- Correlation: Include request/correlation IDs for tracing across services.
- No Sensitive Data: Never log passwords, tokens, or PII.
- Metrics: Track key business and performance metrics.

## 7. Database Migrations
- Versioned: Every schema change requires a migration file.
- Reversible: All migrations must have rollback (down) logic.
- Immutable: Never modify migrations that have been released/deployed.
- Tested: Test rollback locally before deploying.
- Atomic: Each migration should be a single, atomic change.
- No Data Loss: Destructive changes require data backup or migration scripts.

## 8. API Stability
- Backward Compatible: Never break existing clients without versioning.
- Deprecate First: Mark endpoints deprecated before removal (minimum 1 version).
- Versioning: Use URL versioning (`/api/v1/`) from the start.
- Changelog: Document all breaking changes in CHANGELOG.md.
- Contracts: API responses should match documented schemas.
- Graceful Changes: Add new fields, don't rename or remove existing ones.

## 9. Framework Specific Optimization (Laravel Context)
If the project is Laravel, strictly apply these patterns to reduce boilerplate:
* **Request -> DTO -> Service:** Do not pass the raw `Request` object to Services. Extract validated data into a DTO or use `ValidatedRequest` pattern.
* **Guard Clauses via Middleware:** Do not check for user existence or roles inside the controller (e.g., `if (!$user)`). Use Middleware or Policy Gates (`$this->authorize`).
* **Response Classes:** Do not manually build JSON responses (e.g., `response()->json(['status' => true])`). Use a standardized `ApiResponse` class or macros.
* **FormRequest Only:** Never validate input inside the controller method. Always use a dedicated `FormRequest` class.

## 10. Before You Call It Done
Prove the controls work; don't assume them. For each item the change touches:

- **Ownership:** a feature test where a second user requests the first user's record and gets 403 or 404, for read, update and delete.
- **Mass assignment:** a test that sends an extra field (`role`, `is_admin`, `user_id`) and asserts it was not saved.
- **Tenancy:** a test with two tenants that asserts tenant A never sees tenant B's records, in lists as well as single records.
- **Rate limiting:** a test that sends one request over the limit and gets 429.
- **Uploads:** a test that uploads a disallowed type and gets a validation error.
- **Webhooks:** a test with a bad signature (rejected) and a duplicate event (processed once).
- **Grep the diff** for `$request->all()`, `{!! `, `DB::raw(` / `whereRaw(` with variables, `env(` outside `config/`, and `response()->json($model)`.
- Run the full test suite, and report honestly what was and wasn't verified.
