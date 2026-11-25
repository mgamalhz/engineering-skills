---
name: performance
description: Global standards for Scalability and Performance — query efficiency (N+1, eager loading, indexes), pagination, caching and invalidation, async/queue offloading, memory/resource management, and Laravel-specific performance patterns.
---

## 1. Scalability & Throughput
- Efficiency: Avoid nested loops that exceed O(n) complexity. Prefer set-based work (maps, joins, batch queries) over per-item iteration.
- Data: All list-based endpoints must implement pagination and filtering. Never return an unbounded collection.
- Stateless: Services should be stateless so the app scales horizontally. No per-request state on singletons.
- Bounded work: Any operation over a user-growable dataset (all orders, all photos) must be bounded — paginate, chunk, or stream. Assume the table has millions of rows.

> Scope: this skill covers runtime cost (queries, memory, latency, throughput). It does **not** cover code structure/layering/SRP — that's the `architecture` skill — nor trust boundaries — that's the `security` skill.

## 2. Query Performance
- N+1: Never run a query inside a loop over a result set. Eager-load relations up front (`with`, `load`) or batch into a single query.
- Eager loading: Load exactly the relations a code path uses — no more (over-eager loading wastes memory), no less (causes N+1).
- Projection: Select only the columns you need (`select([...])`, `pluck`) instead of hydrating full models when a few fields suffice.
- Existence/counts: Use `exists()`, `count()`, `withCount()` instead of loading a collection just to check `->isEmpty()` or `->count()`.
- Indexes: Every column used in a `WHERE`, `JOIN`, `ORDER BY`, or foreign key must be indexed. Add the index in the same migration that introduces the access pattern.
- Large reads: Iterate large datasets with `chunk`, `chunkById`, `lazy`, or `cursor` — never `->get()` an unbounded table into memory.

## 3. Caching
- Cache expensive computations and frequently-read, rarely-changing queries.
- Always pair a cache write with an explicit invalidation path — a cache you can't invalidate is a bug. Prefer event/observer-driven invalidation over TTL-only when correctness matters.
- Key caches deterministically (include the inputs that vary the result, e.g. locale, user, version). Avoid unbounded key growth.
- Never cache per-user authorization decisions in a shared key.

## 4. Async & Offloading
- Use non-blocking, asynchronous operations for database and network calls where the framework supports it.
- Offload slow or external work (email, image processing, third-party API calls, report generation) to a queue — never block the request/response cycle on it.
- Make queued jobs idempotent and safe to retry; cap retries and define a failure path.
- Batch outbound calls where possible instead of one call per record.
- Deciding WHAT belongs in a transaction vs a queue (business-critical vs side effect) is covered by the `architecture` skill under "Consistency Boundaries".

## 5. Resource Management
- Identify and prevent memory leaks and unbounded memory growth (accumulating arrays/collections across a large loop).
- Stream large responses/files instead of building them fully in memory.
- Release handles, close connections, and avoid holding large objects longer than needed.
- Watch for accidental full-collection materialization (`->get()->filter()` where a `where` clause belongs in SQL).

## 6. Frontend Performance
- Code-split and lazy-load routes/heavy components; keep the initial bundle lean.
- Avoid shipping large dependencies to the client when a smaller one or server-side work suffices.
- Memoize expensive renders/derivations; don't recompute on every render.
- Defer, paginate, or virtualize long lists.

## 7. Framework-Specific (Laravel Context)
Apply these when the project is Laravel:
- **Eager load relations:** `Model::with('relation')` for read paths; `withCount` for counts. Grep smell: an Eloquent access inside a `foreach` over a collection.
- **Chunk/lazy for batch work:** `Model::query()->lazyById()` / `chunkById()` in jobs and commands; never `all()`/`get()` over a growable table.
- **Index in the migration:** add `$table->index([...])` / `$table->foreignId(...)->index()` alongside the column, matching the query's filter/sort.
- **Cache via the cache store:** `Cache::remember(key, ttl, fn)` with a matching `Cache::forget` in the relevant observer/event.
- **Queue the slow stuff:** `dispatch(new Job(...))` for mail, media processing, and external API calls; keep controllers/services returning fast.
- **Paginate:** `->paginate()` / `->cursorPaginate()` on every index endpoint; expose page size limits.
- **Avoid hydrating to count:** `->count()` / `->exists()` at the query builder, not `->get()->count()`.
- **Octane-safe:** no request state stored on container singletons; reset/avoid static accumulation.
