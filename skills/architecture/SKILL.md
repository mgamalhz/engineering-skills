---
name: architecture
description: Use when designing a feature, adding a controller, service, model, job or endpoint, deciding where code belongs, writing anything that changes money, stock, balances, orders or a status, or reviewing a pull request for structure. Covers SOLID, layers, ultra-thin controllers and a service layer, transaction vs queue boundaries, race conditions and row locks, idempotency and status state machines. Language-agnostic principles with Laravel examples. Trust boundaries are the `security` skill; runtime cost is the `performance` skill.
---

## 1. SOLID Principles

### Single Responsibility (SRP)
- Each module/class has ONE reason to change
- Split when a component handles multiple concerns
- Signs of violation: "and" in class names, multiple unrelated methods

### Open/Closed (OCP)
- Open for extension, closed for modification
- Use abstraction to add behavior without changing existing code
- Signs of violation: Frequent edits to switch/case statements

### Liskov Substitution (LSP)
- Subtypes must be substitutable for their base types
- Don't override methods to throw "not implemented"
- Signs of violation: Type checks before method calls

### Interface Segregation (ISP)
- Clients shouldn't depend on methods they don't use
- Split fat interfaces into smaller, focused ones
- Signs of violation: Empty method implementations

### Dependency Inversion (DIP)
- Depend on abstractions, not concrete implementations
- High-level modules shouldn't depend on low-level modules
- Signs of violation: Direct instantiation of dependencies

## 2. Layer Separation

### General Layers
| Layer | Responsibility |
|-------|----------------|
| **Presentation** | Request handling, response formatting |
| **Validation** | Input validation (middleware, request classes, decorators) |
| **Business Logic** | Core rules, orchestration, domain operations |
| **Data Access** | Storage, external APIs, file systems |

### Rules
- Dependencies flow inward: Presentation → Business → Data
- Business layer has zero knowledge of presentation details
- Validation happens BEFORE business logic (via framework conventions)
- Data access is abstracted behind interfaces
- Cross-cutting concerns use middleware/decorators/interceptors

### Validation Placement
- Use framework conventions (Form Requests, DTOs, Decorators, Middleware)
- Keep controllers/handlers thin - just orchestration
- Validate at system boundaries (user input, external APIs)

## 3. Design Patterns

### When to Apply

| Pattern | Use When | Avoid When |
|---------|----------|------------|
| **Factory** | Complex object creation, hiding implementation details | Simple constructors suffice |
| **Strategy** | Multiple interchangeable algorithms | Only one algorithm exists |
| **Observer** | Event-driven communication, loose coupling | Simple direct calls work |
| **Adapter** | Integrating third-party APIs, legacy code | Building from scratch |
| **Decorator** | Adding behavior without modifying original | One-time extension |
| **Facade** | Simplifying complex subsystems | Subsystem is already simple |
| **Singleton** | Single instance resources (config, connection pools) | Just for global access |
| **Builder** | Complex object construction with many options | Few construction parameters |
| **Command** | Undo/redo, queuing operations, logging actions | Simple direct execution |

### Anti-Patterns to Avoid
- **God Object**: One class doing everything
- **Spaghetti Code**: No clear structure or flow
- **Golden Hammer**: Using same pattern for everything
- **Premature Abstraction**: Abstracting before understanding
- **Circular Dependencies**: A depends on B depends on A

## 4. Folder Structure

### Feature-Based (Recommended for large projects)
```
/src
  /features (or /modules, /domains)
    /users
      controller (or handler)
      service
      model (or entity)
      validation (requests, DTOs)
      tests
    /orders
      ...
  /shared (or /common)
    /utils
    /middleware
    /types
  /infrastructure
    /database
    /cache
    /external-apis
```

### Layer-Based (For smaller projects)
```
/src
  /controllers (or /handlers)
  /services
  /models (or /entities)
  /requests (or /validation)
  /utils
  /config
```

### Choosing Structure
- Feature-based: Better for large teams, microservice-ready
- Layer-based: Simpler for small projects, easier onboarding
- Follow framework conventions when they exist

## 5. API Design

### RESTful Conventions
| Action | Method | Endpoint | Response |
|--------|--------|----------|----------|
| List | GET | /resources | 200 + array |
| Get | GET | /resources/:id | 200 or 404 |
| Create | POST | /resources | 201 + created |
| Update | PUT | /resources/:id | 200 + updated |
| Partial | PATCH | /resources/:id | 200 + updated |
| Delete | DELETE | /resources/:id | 204 no content |

### Response Format
- Success responses: `{ data: {...}, meta: {...} }`.
- The error format and API versioning are defined in the `security` skill (Error Handling, API Stability). Follow them there; don't redefine them.

## 6. Dependency Management

### Principles
- Inject dependencies, don't instantiate internally
- Depend on interfaces/abstractions
- Use framework's DI container when available
- External services behind abstractions for testability

## 7. Configuration Management

### Rules
- All configuration via environment variables
- Validate required config at startup (fail fast)
- Never commit secrets or `.env` files
- Use `.env.example` as template

## 8. Module Boundaries

### Public API
- Each module exposes only necessary interfaces
- Use index/barrel files as module entry points
- Keep internal implementations private

### Communication
- Within module: Direct function/method calls
- Across modules: Through defined interfaces
- Across services: Events/messages for loose coupling

## 9. General Guidelines

### Controller/Handler Responsibilities (Laravel examples)
The principle applies to any framework; the rules below name Laravel's tools. In another framework, use its equivalent (request DTOs, serializers, a global error handler).

**Core Philosophy:** The controller is strictly a traffic cop. It accepts input and returns output. No logic allowed.
**Strict Guidelines for Ultra-Thin Controllers:**
1.  **Orchestration Only:** Receive request → Call Service → Return Resource.
2.  **No Exception Handling:** Never use `try-catch` blocks. Let exceptions bubble up to the Global Exception Handler.
3.  **No Data Transformation:** Never manually construct arrays or transform data inside the controller (e.g., do NOT do `$filters = [...]`). Pass the `FormRequest` or `DTO` directly to the Service.
4.  **No Conditional Logic:** Avoid `if/else` for control flow or responses. Use Polymorphism or standardized Response classes.
5.  **Implicit Binding:** Always use Route Model Binding (e.g., `show(Unit $unit)`) instead of manual ID lookups (`find($id)`).
6.  **Strict Output:** return types must be explicit: `JsonResource`, `ResourceCollection`, or `Response` (e.g. 204). Never return raw arrays or `json_encode`.
7. **Ultra Thin**: try to make the controller ultra thin

### Service Layer Requirement
- **ALWAYS use a service layer between controller and model**
- Controllers call services, services call models
- Never access models directly from controllers
- Services contain all business logic, validation orchestration, and logging

### Abstraction Levels
- Each function operates at one abstraction level
- High-level functions orchestrate, low-level functions execute
- Extract when logic is reused OR complex

### Coupling & Cohesion
- Low coupling: Minimal dependencies between modules
- High cohesion: Related things together
- If changing A always requires changing B, they should be together

## 10. Consistency Boundaries (CAP-Informed)

Choose the consistency model per operation, not per system. Ask: "if this step fails, must the whole operation fail?"

| Operation Type | Consistency Model | Implementation |
|----------------|-------------------|----------------|
| **Business-critical writes** (payments, orders, inventory, balances, ownership transfer) | Strong consistency | DB transaction; all-or-nothing; rollback on any failure |
| **Side effects** (email, notifications, analytics, search indexing, media processing, cache warming, webhooks) | Eventual consistency | Dispatch to queue after commit; idempotent, retryable jobs with a defined failure path |

### Rules
- Inside the transaction: only the state changes that must be atomic. Keep transactions short.
- Side effects dispatch AFTER commit (e.g. Laravel `afterCommit`, transactional outbox) so a rollback never leaves a sent email for an order that was never saved.
- Never make external API/HTTP calls inside a DB transaction: it holds locks for the duration and a partial failure leaves inconsistent state.
- Eventual-consistency consumers must tolerate delay and duplicates (idempotency keys, upserts).
- Prefer making after-commit dispatch the default (Laravel queue connection `'after_commit' => true`) over remembering `afterCommit()` on each dispatch.

### Concurrency Inside Transactions (Isolation)
A transaction makes writes all-or-nothing; it does NOT stop two requests from racing. PostgreSQL, Oracle and SQL Server default to Read Committed, which only blocks dirty reads and dirty writes. These bugs pass every test (one request at a time) and show up under load.

- **Read → decide → write is a race.** Two requests both read "coupon available / balance enough / seat free", both decide yes, both write. Close the gap with one of:
  - Lock the row on read: `lockForUpdate()` (`SELECT ... FOR UPDATE`) inside the transaction, then re-check the condition.
  - A single atomic conditional write: `UPDATE ... SET used = true WHERE id = ? AND used = false` and check the affected row count; `increment()`/`decrement()` instead of read-modify-save.
  - A database constraint (unique index, check constraint) as the last line of defense.
- **Related reads that must agree** (a report, totals compared side by side, a backup/export) run in one transaction at `REPEATABLE READ` so every query sees the same snapshot. Separate queries under Read Committed can mix moments (read skew).
- Know your database's default isolation level and what it does not cover; don't assume "ACID" means serializable.

### Idempotent Writes
Clients retry: a double-click, a timeout followed by a retry, a mobile app on a bad network. A write that charges money or creates an order must give the same result when it arrives twice.

- Business-critical create endpoints (payments, orders, transfers, bookings) accept an idempotency key: a client-generated `Idempotency-Key` header, or a natural key such as the cart ID.
- Store the key with a unique index alongside the result. A repeat with the same key returns the first result instead of running again; a repeat while the first is still running gets 409.
- The unique index is what makes it safe. Checking "does this key exist?" in code and then inserting is the same read → decide → write race as above.
- Calls to a payment provider pass the provider's own idempotency key, so a retried job doesn't charge twice.

### Signs of Violation
- `Mail::send` / notification / HTTP call inside `DB::transaction`
- Multi-row business write (order + items + payment) without a transaction
- Job dispatched inside `DB::transaction` without `afterCommit` (the worker can run before commit and hit `ModelNotFoundException`)
- Read a value, check it in PHP, then `save()` it back (coupon use, balance, stock, seat) with no `lockForUpdate`, atomic conditional update or constraint
- `$model->count = $model->count + 1; $model->save();` instead of `increment()`
- A report or dashboard whose related totals come from separate queries outside one snapshot
- A queued job that fails silently with no retry cap or failure handler
- Request latency dominated by side-effect work that the user never needed to wait for
- A payment or order endpoint where submitting the same request twice creates two records or two charges

## 11. Status Fields (State Machines)
A `status` column that any code can set to any value becomes impossible to reason about: orders that are "shipped" but never "paid", or "cancelled" after "delivered".

- List the states and the allowed transitions in one place, next to the model (a PHP enum with a `canTransitionTo()` method, or a state class per status).
- Change the status only through named methods that check the transition: `$order->markAsPaid()`, `$order->cancel()`. Never `$order->update(['status' => ...])` from a controller or a job.
- An illegal transition throws a domain exception; it never silently saves.
- Side effects of a transition (email, stock release, refund) are dispatched by that method or by an event it fires, after commit, so every path that changes the status gets them.
- Record transitions (who, when, from, to) when the history matters to the business.

### Signs of Violation
- `->update(['status' => ...])` or `->status = ...` in more than one place
- `if ($order->status === 'paid' || $order->status === 'shipped')` checks spread across controllers and views
- Status values as raw strings instead of an enum

## 12. Before You Call It Done
Check the structure of what you wrote, not just that it works:

- **Controllers:** each action receives a `FormRequest` or bound model, calls one service method, and returns a Resource or a `Response`. No `try/catch`, no queries, no `if/else` on business rules.
- **Transactions:** grep the diff for `DB::transaction` and read every block: no `Mail::`, `Notification::`, `Http::`, `dispatch(` without `afterCommit`, or other external calls inside it.
- **Races:** for every read → decide → write on money, stock, seats, coupons or balances, point to the lock, atomic update or unique index that protects it.
- **Idempotency:** a feature test that sends the same create request twice and asserts one record.
- **Status changes:** a test that an illegal transition throws, for every transition you added.
- Report what you verified and what you didn't.
