# Claude skills

The engineering standards I give my coding agent, written as
[Agent Skills](https://agentskills.io). They make the agent write code the way
I review it: thin controllers, safe transactions, no N+1 queries and no
secrets in code.

They work with Claude Code, and with any agent that reads the Agent Skills
format (Cursor, Codex and others).

| Skill | What it gives the agent |
|---|---|
| [`architecture`](#architecture) | Where code lives and how parts depend on each other: SOLID, layers, thin controllers, service layer, and when to use a transaction vs a queue. |
| [`security`](#security) | Trust boundaries plus delivery standards: secrets, input validation, auth, testing, error handling, logging, migrations and API stability. |
| [`performance`](#performance) | Runtime cost: N+1 queries, indexes, pagination, caching with invalidation, queues and memory. |

The three skills are written to not overlap. `architecture` decides where code
goes, `security` guards what crosses a trust boundary, and `performance`
watches what the code costs at runtime. They point to each other instead of
repeating the same rules.

The rules are language-agnostic. Where a framework needs exact guidance,
`security` and `performance` have a **Laravel** section with the specific APIs
to use, and `architecture` gives its examples in Laravel.

## Install

**Claude Code** (as a plugin)

```bash
/plugin marketplace add mgamalhz/claude-skills
/plugin install engineering@mgamalhz
```

**Any agent** (via [skills.sh](https://skills.sh))

```bash
npx skills add mgamalhz/claude-skills
npx skills add mgamalhz/claude-skills --skill security   # just one
```

**Manually:** copy a folder from `skills/` into your agent's skills directory:
`~/.claude/skills/` to use it in every project, or `.claude/skills/` inside one
project.

Once installed, you don't call the skills by hand. The agent reads each skill's
`description` and loads it when the task matches, for example `performance`
when you build a list endpoint, or `security` when you add an endpoint that
takes user input.

---

## architecture

**Use it for:** designing a feature, reviewing a pull request, or deciding where
a piece of code belongs.

What it covers:

- **SOLID**, with the signs that each principle is being broken (an "and" in a
  class name, type checks before method calls, empty interface methods).
- **Layers.** Dependencies flow one way: presentation → business logic → data
  access. Validation happens at the boundary, before business logic runs.
- **Ultra-thin controllers.** A controller receives the request, calls a service
  and returns a resource. No `try/catch`, no data transformation, no `if/else`,
  route model binding instead of `find($id)`, and explicit return types.
- **A service layer, always.** Controllers never touch models directly.
- **Design patterns**, with when to use each one and when not to, plus the
  anti-patterns to avoid (God object, premature abstraction, circular
  dependencies).
- **Folder structure** (feature-based vs layer-based), REST conventions, a
  consistent response format, dependency injection and configuration.
- **Consistency boundaries.** For each operation it asks one question: if this
  step fails, must the whole operation fail?
  - Business-critical writes (payments, orders, stock, balances) go in a DB
    transaction.
  - Side effects (email, notifications, webhooks, indexing) go to a queue,
    dispatched **after commit**.
  - No HTTP calls inside a transaction.
- **Concurrency inside transactions.** A transaction is all-or-nothing, but it
  does not stop two requests from racing. The skill shows how to close the
  "read → decide → write" gap with `lockForUpdate()`, an atomic conditional
  `UPDATE`, or a database constraint, and when a report needs
  `REPEATABLE READ`.

What it catches, for example:

```php
// ✗ The email is sent even if the order rolls back, and the HTTP call holds locks.
DB::transaction(function () use ($order) {
    $order->save();
    Mail::to($order->user)->send(new OrderPlaced($order));
});

// ✗ Two requests can both read "unused" and both use the coupon.
if (! $coupon->used) {
    $coupon->used = true;
    $coupon->save();
}
```

## security

**Use it for:** any code that handles user input, auth, secrets or data, and as
the checklist before calling work done.

What it covers:

- **Security:** no hardcoded secrets, schema validation for every input,
  parameterized queries, auth and authorization on every protected endpoint,
  security headers, and dependency audits.
- **Code quality:** SOLID and DRY, short self-documenting functions, strict
  types, no dead code or leftover logs.
- **Testing:** 80% coverage on business logic, unit tests for public functions,
  integration tests for endpoints, and edge cases (empty states, boundaries,
  failures).
- **Error handling:** one error format, `{ error: { code, message, details? } }`,
  no raw errors leaking to clients, and no silent failures.
- **Logging:** structured JSON logs with correlation IDs, and never passwords,
  tokens or personal data in logs.
- **Migrations:** versioned, reversible, never edited after release, and no
  destructive change without a backup.
- **API stability:** versioning from day one, deprecate before removing, add
  fields instead of renaming them.
- **Laravel:** `FormRequest` for validation, request → DTO → service, policies
  and middleware instead of role checks in controllers, and a standard response
  class.

## performance

**Use it for:** list endpoints, large tables, reports, relationships, caching,
queues and anything that grows with your data.

What it covers:

- **Scale:** every list is paginated, every operation over growing data is
  bounded (paginate, chunk or stream), and services are stateless. It assumes
  the table has millions of rows.
- **Queries:** no query inside a loop (eager load instead), load exactly the
  relations a code path uses, select only the needed columns, use
  `exists()`/`count()` instead of loading a collection, and index every column
  used in `WHERE`, `JOIN` or `ORDER BY` in the same migration.
- **Caching:** every cache write has an invalidation path, keys include
  everything that changes the result, and per-user permissions are never
  cached under a shared key.
- **Queues:** slow or external work (mail, media, third-party APIs, reports)
  never blocks the request. Jobs are idempotent, with capped retries and a
  failure path.
- **Memory and frontend:** stream large files, avoid loading whole collections,
  code-split routes and virtualize long lists.
- **Laravel:** `with()` and `withCount()`, `lazyById()`/`chunkById()` in jobs,
  `paginate()`/`cursorPaginate()` on every index, `Cache::remember` with a
  matching `Cache::forget`, and Octane-safe singletons.

What it catches, for example:

```php
// ✗ One query per order (N+1), and the whole table loaded into memory.
foreach (Order::all() as $order) {
    echo $order->customer->name;
}

// ✓ Two queries, in pages.
Order::with('customer')->paginate();
```

---

## License

MIT
