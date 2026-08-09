# Production Notes

## Routing fresh-aggregate reads to a read replica

`withFreshAggregates()` runs an aggregation per outer row — on a balanced-fanout tree at N=10K it's the most expensive read the package emits. If you have read replicas, route these reads off the primary:

```php
Category::query()
    ->withFreshAggregates()
    ->useReadPdo()        // ← stays on Laravel's read connection
    ->get();
```

Caveat: Eloquent automatically routes any query inside an open transaction to the **write** PDO regardless of `useReadPdo()`, to avoid replication-lag visibility issues. If you wrap the read in a transaction (or call it from inside one), it lands on the primary anyway. For genuine replica routing, the read needs to live outside a transaction boundary.

Pair with the `nestedset.aggregate_locking` config flag — `'never'` is safe on a read-only path; the locking modes only matter for the write path.

## MariaDB: disabling `split_materialized`

The fresh-aggregate read path uses a derived-table JOIN on MariaDB so the subquery is materialised once per outer query rather than once per row. MariaDB's optimizer can convert that derived JOIN into a LATERAL DERIVED via `split_materialized`, which collapses the materialise-once advantage and runs \~3× slower in practice. `withMariaDbSplitMaterializedOff()` prepends a `SET STATEMENT optimizer_switch='split_materialized=off' FOR …` to the next compiled SQL — scoped to the one statement, no session-state mutation:

```php
Category::query()
    ->withFreshAggregates()
    ->withMariaDbSplitMaterializedOff()
    ->get();
```

No-op on MySQL/PostgreSQL/SQLite — the `SET STATEMENT` prefix is MariaDB-specific syntax. Only reach for it if profiling shows the fresh-aggregate path running unexpectedly slow on MariaDB.

> [!NOTE]
> The flag is consulted inside `Builder::runSelect()` only — i.e. it takes effect on `->get()` / `->first()` / `->paginate()` / `->chunk()` (all of which funnel through `get()` → `runSelect()`). Calls that bypass `runSelect()` — `->cursor()` (calls `Connection::cursor()` directly), aggregate methods (`->count()`, `->sum()`, `->exists()`), and DML (`->update()`, `->delete()`) — don't carry the prefix. In practice this matches the use case — the fresh-aggregate overhead is on the row-returning paths — but worth knowing if you profile a `->count()` (or stream via `->cursor()`) and wonder why the optimisation didn't show up. For cursor-style streaming over a derived-shape fresh-read query on MariaDB, wrap the read in a `chunk()` instead.

## Telemetry

The package fires typed events on Laravel's event bus around every meaningful operation — bulk insert lifecycle, structural moves, cascade soft-delete / restore / force-delete, tree repair, aggregate maintenance, scope-violation guards. See the dedicated [Events](events.html) page for the full catalogue, payload tables, queue-safety guidance, and worked recipes (search indexing, cache invalidation, audit logs, Datadog / Sentry wirings).

Quick links:

- **Telemetry events** (scalar payloads, safe for `ShouldQueue`): `FixTreeCompleted`, `FixAggregatesCompleted`, `FixAggregatesChunkCompleted`, `FixAggregatesJobDispatched`, `DeferredAggregateMaintenanceCompleted`, `DeferredMaintenanceStarting`, `TreeIntegrityChecked`, `AggregateDriftDetected`, `NodeAggregatesRecomputed`, `NodeMoved`, `ScopeViolationDetected`, `BulkInsertTreeCompleted`.
- **Model-carrying events** (live Eloquent instances, not queue-safe by default): `BulkInsertTreeStarting`, `BulkInsertTreePlanned`, `BulkInsertNodeSaved`, `BulkInsertTreeSaved`, every `Subtree*` event, `NodesSwapped`, `NodePromotedToRoot`, `SoftDeleteMarkerCaptured`.
- **Exception-carrying** (not queue-safe — `Throwable` won't serialise across most drivers): `AggregateMaintenanceFailed`. Capture `$e->stage` / `$e->modelClass` synchronously and forward those if you want a queued listener.

Set `nestedset.events_enabled => false` to short-circuit every firing site. Default is `true`.
