# Architecture Overview

This section is a source-level walkthrough of how `vusys/laravel-nestedset` is built — the algorithms, the SQL, and how the classes collaborate. The task-oriented sections (Tree Operations, Querying, Aggregates, Maintenance) tell you *what the package does*; Internals tells you *how it does it*, so you can read the source, debug a corruption report, or contribute with confidence.

> [!NOTE]
> Citations name the file plus the `Class::method()` — those are the durable anchors. Line numbers are a convenience and drift as the code changes. This walkthrough was captured against **`v0.13.0-26-gca9b1fb`**; if your checkout differs, trust the method names over the line numbers.

## The shape of the package

Every public capability is reachable from one trait, `NodeTrait` (`src/NodeTrait.php`). A model opts in by using the trait and implementing the `HasNestedSet` contract:

```php
use Illuminate\Database\Eloquent\Model;
use Vusys\NestedSet\Contracts\MaintainsTreeAggregates;
use Vusys\NestedSet\NodeTrait;

class Category extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;
}
```

`NodeTrait` is deliberately thin — it is a **composition of eleven concerns**, each owning one slice of behaviour, plus a handful of Eloquent overrides. The trait body is little more than a list of `use` statements:

```php
trait NodeTrait
{
    use HasBulkInsert;
    use HasMaterialisedPath;
    use HasNestedSetAggregates;
    use HasNodeInspection;
    use HasSoftDeleteTree;
    use HasSubtreeClone;
    use HasTreeExport;
    use HasTreeMutation;
    use HasTreeRelations;
    use HasTreeRepair;
    use HasTreeWalk;
}
```

| Concern (`src/Concerns/`) | Owns                                                                                                                                                                 | Walkthrough                                                      |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `HasTreeMutation`         | `appendToNode` / `prependToNode` / `insertBeforeNode` / `insertAfterNode` / `makeRoot` / `moveTo` / `up` / `down` — queues a `PendingOperation`, flushed on `save()` | [The Mutation Engine](mutation-engine.html)                      |
| `HasTreeRelations`        | `parent`, `children`, and the custom eager-loadable `ancestors` / `descendants` relations                                                                            | [Query Engine & Relations](query-engine.html)                    |
| `HasNodeInspection`       | `isRoot` / `isLeaf` / `isDescendantOf` / `getSubtreeSize` — pure in-memory predicates over the bounds                                                                | [The Nested-Set Model](nested-set-model.html)                    |
| `HasTreeRepair`           | `isBroken` / `countErrors` / `fixTree` and the `TreeFixResult` value object                                                                                          | [Integrity & Repair](repair.html)                                |
| `HasNestedSetAggregates`  | precalculated rollup columns (SUM/COUNT/AVG/MIN/MAX, filtered, listener-based)                                                                                       | [Aggregate Maintenance](aggregate-maintenance.html)              |
| `HasSoftDeleteTree`       | cascade soft-delete / restore with `deleted_at` stamp matching                                                                                                       | [Aggregate Maintenance](aggregate-maintenance.html#soft-deletes) |
| `HasBulkInsert`           | `bulkInsertTree()` — one `makeGap` + N saves + one deferred `fixAggregates`                                                                                          | [Bulk Insertion](../tree-operations/bulk-insertion.html)         |
| `HasTreeExport`           | `toAsciiTree()` / `toMermaid()` / `toDot()` / `toJsonTree()` tree serialisers, plus `*Forest` / `*Scope` static counterparts                                         | [Tree Exporters](../querying/exporters.html)                     |
| `HasTreeWalk`             | `walk()` / `dfs()` / `dfsPostOrder()` / `bfs()` / `flattenedSubtree()` — visitor + generators over a loaded subtree, with `WalkContext` and `WalkFilter`             | [Walking Subtrees](../querying/walking.html)                     |
| `HasSubtreeClone`         | `cloneSubtreeTo()` / `cloneSubtreeAsRoot()` — deep-copy a subtree (refresh + delegate to `bulkInsertTree`)                                                           | [Cloning Subtrees](../tree-operations/cloning.html)              |
| `HasMaterialisedPath`     | maintained slug/id path columns alongside `lft`/`rgt`                                                                                                                | [Materialised Paths](../tree-operations/materialised-paths.html) |

### The walker — `src/Walker/`

`HasTreeWalk` is thin glue; the real implementation lives in four value objects:

- **`SubtreeWalker`** is the engine. It takes a flat collection plus a root model, builds two indexes on construction (`byKey: id => Model` and `childrenByParentKey: parentId => list<id>`, both `O(N)`), and exposes iterative-DFS and queue-BFS generators that yield `(Model, WalkContext)` tuples. Both DFS strategies use an explicit task stack so deep trees do not blow PHP's call stack. The walker also implements `Countable` and exposes `maxDepth()` / `leafCount()` companions — all three share a single-pass index walk and memoise.
- **`WalkContext`** is the readonly value object passed as the visitor's second argument. It carries depth (relative to the walk root, not the absolute `depth` column), parent, sibling index/count, the derived `isFirstSibling`/`isLastSibling` flags, and a lazy `pathToRoot()` that walks up via the index on first call.
- **`WalkFilter`** is a `final readonly` triple of `maxDepth`, `visitable`, `includeRoot`. Named constructors (`depth()`, `where()`, `compose()`) and an instance `andThen()` cover the common shapes. The same `WalkFilter` plugs into every exporter's option object so a depth/predicate filter applied to ASCII output works identically against Mermaid and JSON.
- **`WalkSignal`** is a two-case enum (`SkipSubtree`, `Stop`) returned by visitor closures to steer the walk. Skip is honoured by pre-order DFS and BFS, ignored by post-order. Returning `null` (or `void`) continues.

The walker is purely a consumer of in-memory data — it never queries. When the public methods on `HasTreeWalk` are called without an explicit `$subtree`, they fall back to `$this->descendants`; if the relation is not loaded either, they throw `UnloadedSubtreeException`. The exporters use it internally to compute their visible-key set when a `WalkFilter` is supplied.

Models **must** `implements MaintainsTreeAggregates` (`src/Contracts/MaintainsTreeAggregates.php`, which extends `HasNestedSet` in `src/Contracts/HasNestedSet.php`). The trait supplies a default implementation of every contract method, so the interface costs nothing to satisfy — its job is to give Larastan (and your IDE) a typed surface to resolve `getLft()`, `getBounds()`, and the column-name accessors against.

## The layers underneath

The concerns are the API surface. They delegate the actual SQL to a layer of query builders under `src/Query/`:

```text
 model (NodeTrait)
   │  appendToNode($parent)->save()
   ▼
 HasTreeMutation            ← queues a PendingOperation, dispatches on save()
   │  newTreeMutator()
   ▼
 TreeMutationBuilder        ← emits the atomic CASE WHEN UPDATE (makeGap/moveNode)
   │  new TreeExpression(...)
   ▼
 TreeExpression             ← thin Expression wrapper for package-owned SQL fragments
   ▼
 database
```

- `TreeBaseQueryBuilder` — the package's `Illuminate\Database\Query\Builder` subclass; a home for SQL-execution hooks (e.g. the MariaDB `optimizer_switch` prefix used by fresh-aggregate reads).
- `TreeQueryBuilder` — the Eloquent builder returned from every `NodeTrait` model. Adds `whereDescendantOf`, `whereAncestorOf`, `whereIsRoot`, `withDepth`, `defaultOrder`, `withFreshAggregates`, and more.
- `TreeMutationBuilder` / `TreeRepairBuilder` — internal builders for the write and repair paths. They take a connection, table, and column names and emit the single-statement updates.
- `TreeExpression` — a thin `Expression` wrapper that lets dynamically-composed (but package-owned) SQL bypass Laravel's `literal-string` constraint, so the query builder splices it into the `SET` clause verbatim. It is **not** a backend dispatcher: the per-driver SQL shapes (LATERAL on PostgreSQL/MySQL, derived-table on MariaDB, correlated fallback on SQLite) live in `src/Query/Aggregates/Read/AggregateSqlFragments.php` and `FreshAggregateProjector.php`.

`NodeTrait` wires these in via two Eloquent overrides:

```php
public function newEloquentBuilder($query): TreeQueryBuilder
{
    return new TreeQueryBuilder($query);
}

protected function newBaseQueryBuilder(): TreeBaseQueryBuilder
{
    $connection = $this->getConnection();

    return new TreeBaseQueryBuilder(
        $connection,
        $connection->getQueryGrammar(),
        $connection->getPostProcessor(),
    );
}
```

Because `newEloquentBuilder()` is narrowed to return `TreeQueryBuilder`, every `Category::query()` chain exposes the tree scopes with full static analysis — no macros, no `@method` annotations.

## Lifecycle wiring — `bootNodeTrait()` {#lifecycle-wiring}

The trait hooks into the Eloquent model lifecycle in `bootNodeTrait()` (`src/NodeTrait.php`). This single method is the spine of the whole package: the mutation engine, the aggregate maintenance, and the cascade logic all hang off these standard model events.

| Eloquent event | Hook                                                                                           | What it does                                                                                                                                                                                                          |
| -------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `saving`       | `callPendingAction()` then `captureAggregateDeltas()`                                          | Dispatch the queued structural mutation; for existing rows, snapshot the aggregate deltas before the row updates. Also throws `UnplacedNodeException` if a new row reaches `save()` without being placed in the tree. |
| `saved`        | `applyAggregateDeltas()`                                                                       | Issue the captured aggregate `UPDATE` up the ancestor chain.                                                                                                                                                          |
| `created`      | `applyAggregateOnCreate()`                                                                     | Push a freshly inserted node's contribution to its ancestors.                                                                                                                                                         |
| `deleting`     | re-read structural columns                                                                     | Reload `lft`/`rgt`/`depth`/`parent_id` (+ scope columns) from the DB so the cascade and gap-close act on current values, not a stale in-memory copy.                                                                  |
| `deleted`      | soft cascade → force cascade → `applyAggregateOnDelete()` → `applyStructuralCleanupOnDelete()` | Cascade the delete through descendants, decrement ancestor aggregates, then close the lft/rgt gap.                                                                                                                    |
| `restored`     | restore cascade → `applyAggregateOnRestore()`                                                  | Un-trash descendants, then re-add their contribution to ancestors.                                                                                                                                                    |

Each aggregate hook runs inside `runAggregateHook()`, a `try`/`catch` that dispatches an `AggregateMaintenanceFailed` event before re-throwing — so a failing rollup surfaces to Sentry/Bugsnag while still rolling back the wrapping transaction.

A detail worth internalising early: the `saving` hook guards against the most common footgun — calling `Model::create([...])` or `->save()` on a node that was never placed in the tree:

```php
if (! $node->exists
    && method_exists($node, 'isPlacedInTree')
    && ! $node->isPlacedInTree()
) {
    throw new UnplacedNodeException(/* ... */);
}
```

Without this, the row would land with `lft = rgt = 0` (the migration default) and create an `invalid_bounds` corruption. The package fails loudly instead.

## The aggregates subsystem

`src/Aggregates/` is a self-contained subsystem the lifecycle hooks call into. There are three kinds of aggregate column, all sharing the same hooks:

- **SQL aggregates** — declared via `#[NestedSetAggregate]`; SUM/COUNT/AVG/ MIN/MAX (and variance, bitwise, bool, geometric/harmonic mean, …) over a source column with optional filters.
- **Listener aggregates** — declared via `#[NestedSetAggregateListener]`; a PHP `contribution(Model $node)` method computes each row's value.
- **Ad-hoc fresh aggregates** — `withFreshAggregates([...])`; no stored column, recomputed on read via a correlated subquery.

Maintenance picks one of two strategies per column — `Strategy/DeltaMaintenance` (a signed delta applied to ancestors) or `Strategy/RecomputeMaintenance` (a `SELECT`-then-`UPDATE` of the invalidated subset) — depending on whether the change can be expressed as a delta. The full machinery is covered in [Aggregate Maintenance](aggregate-maintenance.html).

## Scoping (multi-tree tables)

A single table can hold many independent trees ("scopes"). Declare the partition column(s) with `#[NestedSetScope('menu_id')]` or a `getScopeAttributes()` method. `NestedSetScopeResolver` (`src/Scope/`) derives the scope; every builder constrains its writes to the scope, and `HasTreeMutation` rejects cross-scope writes with `ScopeViolationException`. Scoped repair methods (`fixTree`, `fixAggregates`) require an anchor node so a repair stays inside one tree rather than walking a multi-million-row table. See [Scoped Trees](../querying/scoped-trees.html).

## The service provider — schema macros {#service-provider}

`NestedSetServiceProvider` (`src/NestedSetServiceProvider.php`) registers four `Blueprint` macros plus a per-connection hook:

- `$table->nestedSet(scope, cover, parentIdType)` — adds the four columns (`lft`/`rgt` unsigned bigint, `parent_id` nullable, `depth` unsigned int) and one composite index.
- `$table->dropNestedSet(...)` — the inverse.
- `$table->nestedSetAggregate('column', type: ...)` — adds an aggregate storage column **plus its companion columns** (see [Aggregate Maintenance](aggregate-maintenance.html#companion-columns)).
- `$table->dropNestedSetAggregate(...)` — the inverse, dropping companions too.

The composite index column order is set by `nestedSetIndexColumns()`:

```php
return [
    ...self::toColumnList($scope),   // scope columns first
    $lft,
    $rgt,
    $parentId,
    ...self::toColumnList($cover),   // covering columns last
];
```

Scope first means each tree occupies its own contiguous slice of the index; the `cover` tail lets subtree-aggregate subqueries (`WHERE inner.lft >= outer.lft AND inner.rgt <= outer.rgt`) run as covering scans with no heap visits. The provider also installs SQLite user-defined `BIT_OR`/`BIT_AND`/`BIT_XOR` aggregates on every fresh connection so the same native-aggregate SQL runs on all four backends.

## Configuration

`config/nestedset.php` is small and worth reading in full once:

- `columns.*` — override the four structural column names.
- `auto_transaction` (default `true`) — wrap every mutation in a transaction.
- `aggregate_locking` (`auto` / `always` / `never`) — control `FOR UPDATE` locking on the recompute path. See [Concurrency & Transactions](concurrency.html).
- `queue.*` — connection/queue routing for `queueFixAggregates()`.
- `events_enabled` (default `true`) — short-circuit every telemetry firing site.

## Load-bearing design decisions

These five choices explain most of *why* the code looks the way it does. Keep them in mind while reading the rest of this section.

### `parent_id` is the source of truth

`lft`/`rgt`/`depth` are a derived index. `fixTree()` rebuilds the index from a `parent_id` walk — never the reverse. A corrupted index is always recoverable as long as `parent_id` is intact. → [Integrity & Repair](repair.html)

### Every shift is one atomic `CASE WHEN UPDATE`

Inserts, moves, and bulk repairs renumber many rows in a single statement, so the on-disk state never passes through an invariant-violating intermediate. → [The Mutation Engine](mutation-engine.html)

### `depth` is stored, not computed

It is maintained on every mutation, which makes level queries and aggregate ancestry scans cheap.

### Mutations are queued, then dispatched on `save()`

`appendToNode($parent)` only records a `PendingOperation`; the write happens in the `saving` hook, so the gap, the INSERT/UPDATE, and the aggregate hooks share one transaction.

### Telemetry is observable but optional

Every meaningful operation fires a typed event, gated by `events_enabled` and a per-event listener check so an unobserved event costs nothing. → [Concurrency & Transactions](concurrency.html#event-dispatch)

## Where to go next

Read [The Nested-Set Model](nested-set-model.html) first — it establishes the encoding and invariants every other page assumes. From there, the pages can be read in any order.
