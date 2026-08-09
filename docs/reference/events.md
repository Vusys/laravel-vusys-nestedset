# Events

The package fires typed events on Laravel's event bus around every meaningful operation. Listen via standard `Event::listen()` to wire metrics (Datadog, New Relic, OpenTelemetry), errors (Sentry, Bugsnag), audit logs, search-index updates, or cache invalidation.

All event classes live under `Vusys\NestedSet\Events\` and are `final readonly` value objects.

## Two flavours

Events come in two distinct flavours, and the distinction matters when you wire listeners:

**Telemetry events** carry only scalar / array payloads — model class, ids, durations, counts. They're safe to subscribe to with `ShouldQueue` listeners and are aimed at metrics, alerting, and observability.

**Model-carrying events** carry live Eloquent model instances and / or descendant-id lists. They're aimed at in-process work that needs the actual data — bulk indexing, decoration, cache priming, subtree-wide invalidation. **Not queue-safe by default**: capture the fields you care about synchronously and forward those, or accept that the listener runs in the request that fired the event.

## Toggling events off

Set `nestedset.events_enabled => false` in `config/nestedset.php` to short-circuit every firing site in the package. Useful only on genuinely hot paths where you've measured the cost of constructing event objects you'll never observe. Default is `true`.

## Cost model

Telemetry events are essentially free — they construct a readonly value object and call `event()`.

Model-carrying events that include descendant ids (`SubtreeSoftDeleted`, `SubtreeRestored`, `SubtreeForceDeleting/Deleted`, `SubtreeMoved`) do **one extra SELECT** to collect the ids — but only when a real listener is registered (`Event::hasListeners(EventClass)`). With no subscribers the firing site short-circuits and no SELECT runs. The check uses Laravel's dispatcher, so it picks up listeners registered via `EventServiceProvider`, `Event::listen()`, and `EventSubscriber`s identically.

## Catalogue

### Bulk insert lifecycle

`Model::bulkInsertTree()` emits five events in order. Subscribe to the one whose granularity matches your use case.

| Event                     | Fires when                                                               | Payload                                                           |
| ------------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------- |
| `BulkInsertTreeStarting`  | top of the call, before plan walk                                        | `modelClass`, `appendTo`, raw `$tree`                             |
| `BulkInsertTreePlanned`   | after the DFS plan walk, before the transaction                          | `modelClass`, `appendTo`, flat `$plan` with relative bounds       |
| `BulkInsertNodeSaved`     | once per row inside the save loop                                        | `modelClass`, `node`, `planIndex`, `totalNodes`, `parent`         |
| `BulkInsertTreeSaved`     | after every row is saved AND the closing `fixAggregates` has run         | `modelClass`, `anchorId`, `appendTo`, `list<Model&HasNestedSet>`  |
| `BulkInsertTreeCompleted` | immediately after `BulkInsertTreeSaved`, with id summary for queued work | `modelClass`, `anchorId`, `rowsInserted`, `durationMs`, `nodeIds` |

`BulkInsertTreeSaved` is the headline event for in-process indexing — you get every saved model in one go, in DFS pre-order. Stored aggregate columns *in the database* are fully rolled up by the time this fires, but the in-memory `$nodes` array was captured during the save loop before the repair pass ran, so the model instances carry their pre-roll-up aggregate values. Call `->fresh()` on a node (or re-query by `$nodeIds` from `BulkInsertTreeCompleted`) to read the final values.

### Cascade events (soft-delete / restore / force-delete)

When the package cascades soft-delete, restore, or hard-delete through a subtree, it issues a single SQL statement against descendants — the per-row Eloquent `deleted` / `restored` events **never fire** for those descendants. These events close that gap.

| Event                      | Fires when                                                                        | Payload                                      |
| -------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------- |
| `SubtreeSoftDeleting`      | before the cascade UPDATE that propagates `deleted_at`                            | `anchor`, `bounds`, `scope`, `deletedAt`     |
| `SubtreeSoftDeleted`       | after the cascade UPDATE                                                          | adds `descendantIds`                         |
| `SubtreeRestoring`         | before the restore-cascade UPDATE                                                 | `anchor`, `bounds`, `scope`, `marker`        |
| `SubtreeRestored`          | after the restore-cascade UPDATE                                                  | adds `descendantIds`                         |
| `SubtreeForceDeleting`     | before the hard-delete cascade on `forceDelete()` of an interior node             | `anchor`, `bounds`, `scope`, `descendantIds` |
| `SubtreeForceDeleted`      | after the cascade DELETE                                                          | adds `descendantsAffected`                   |
| `SoftDeleteMarkerCaptured` | inside `restoring`, when the package records the marker used to match descendants | `anchor`, `marker`                           |

`descendantIds` is the *strict* descendant set (excludes the anchor). The anchor itself fires Eloquent's normal `deleted` / `restored` events in addition to the `Subtree*` pair.

### Subtree cloning

`cloneSubtreeTo()` / `cloneSubtreeAsRoot()` suppress per-row Eloquent `creating` / `created` / `saving` / `saved` events for cloned rows (they ride on top of `bulkInsertTree`). The single signal listeners hook is `SubtreeCloned`.

| Event           | Fires when                                                                                                    | Payload                                                       |
| --------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `SubtreeCloned` | after a successful `cloneSubtreeTo()` / `cloneSubtreeAsRoot()` — deferred to the outermost transaction commit | `modelClass`, `source`, `clone`, `rowCount`, `includeTrashed` |

`source` is the original (re-read) root; `clone` is the new root of the cloned subtree. `rowCount` is the number of rows actually materialised on the destination side — equal to the source subtree size when `includeTrashed` is `true`, otherwise reduced by any trashed descendants that were silently skipped. Not queue-safe: it carries live model instances.

### Subtree movement

| Event       | Fires when                                                                                                   | Payload                                                       |
| ----------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------- |
| `NodeMoved` | structural mutation of an existing node (one event per move; `up()`/`down()` fire two — one per participant) | `nodeId`, `fromBounds`, `toBounds`, `operation`, `durationMs` |

`NodeMoved.operation` is one of `'appendTo'`, `'prependTo'`, `'sibling'`, `'root'`, or `'sibling-displaced'`. The first four mirror `PendingOperation::$action` — the participant the caller actually asked to move. `'sibling-displaced'` identifies the *other* participant in an `up()`/`down()` swap: the sibling that was shifted to make room. `switch ($e->operation)` consumers must include a case for `'sibling-displaced'` (or a default branch), or every `up()`/`down()` will silently miss half its events. | `SubtreeMoving` | before the structural SQL for an existing-node mutation | `anchor`, `fromBounds`, `operation` | | `SubtreeMoved` | after the structural SQL completes | `anchor`, `fromBounds`, `toBounds`, `operation`, `descendantIds`, `durationMs` | | `NodesSwapped` | `up()` / `down()` sibling swap completes | both participants + before/after bounds + `direction` | | `SiblingsReordered` | `reorderChildren()` / `moveToSiblingPosition()` / `reorderChildrenBy()` non-identity reorder completes | `parent`, `idsInOrder`, `rowsAffected`, `durationMs` | | `NodePromotedToRoot` | `makeRoot()` on an existing node | `anchor`, `previousParentId`, `previousDepth` |

The `NodeMoved` / `SubtreeMoved` pair exists because moving an interior node renumbers its entire subtree in SQL — `NodeMoved` carries only the anchor's bounds, so listeners that need the whole moved subtree (breadcrumbs, permission caches, search indexes that key on ancestor paths) should subscribe to `SubtreeMoved` for the descendant-id list.

### Tree repair

| Event                  | Fires when                                | Payload                                                                   |
| ---------------------- | ----------------------------------------- | ------------------------------------------------------------------------- |
| `FixTreeCompleted`     | `Model::fixTree()` finishes               | `modelClass`, `anchorId`, `nodesUpdated`, `durationMs`, `aggregatesFixed` |
| `TreeIntegrityChecked` | every `isBroken()` / `countErrors()` call | `modelClass`, `anchorId`, `errors`, `totalErrors`                         |

`TreeIntegrityChecked` fires on every check, including clean trees — useful as a heartbeat for monitoring. Filter on `$totalErrors > 0` if you only want alerts when drift exists.

### Aggregate maintenance

| Event                                   | Fires when                                                                                                                                                                                    | Payload                                                                               |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `FixAggregatesCompleted`                | `Model::fixAggregates()` finishes (sync, single-shot or chunked)                                                                                                                              | `anchorId`, `totalRowsUpdated`, `perColumn`, `durationMs`, `chunkSize`, `totalChunks` |
| `FixAggregatesChunkCompleted`           | once per chunk in sync chunked + per dispatch in queued chunked                                                                                                                               | `anchorId`, `chunkIndex`, `chunkSize`, `rowsUpdated`, `cursorAfter`, `durationMs`     |
| `FixAggregatesJobDispatched`            | `Model::queueFixAggregates()` hands a job to the dispatcher                                                                                                                                   | `anchorId`, `chunkSize`, `onConnection`, `onQueue`                                    |
| `DeferredMaintenanceStarting`           | outermost entry of `withDeferredAggregateMaintenance()`                                                                                                                                       | `anchorId`                                                                            |
| `DeferredAggregateMaintenanceCompleted` | outermost exit of `withDeferredAggregateMaintenance()` after the closing repair                                                                                                               | `anchorId`, `rowsFixed`, `closureDurationMs`, `repairDurationMs`                      |
| `NodeAggregatesRecomputed`              | once per lifecycle hook (`on_create` / `on_delete` / `on_restore` / `move`) when the model declares aggregates                                                                                | `nodeId`, `columns`, `stage`                                                          |
| `NestedSetAggregateChanged`             | per-row, per-column CDC-style diff — fires once for every (ancestor row, aggregate column) pair whose stored value actually moved during a maintenance pass. **Opt-in by listener presence.** | `nodeId`, `column`, `oldValue`, `newValue`, `ancestorChain`, `stage`                  |
| `AggregateDriftDetected`                | `aggregateErrors()` finds at least one column with non-zero drift                                                                                                                             | `anchorId`, `perColumn`, `totalDrift`                                                 |
| `AggregateMaintenanceFailed`            | exception escapes one of the trait's aggregate-maintenance hooks; the original is rethrown                                                                                                    | `anchorId`, `stage`, `exception`                                                      |

`NodeAggregatesRecomputed` is the cache-invalidation signal: when an aggregate column on the ancestor chain has just been recomputed for this node, invalidate cached rollups under the same ancestor scope.

`NestedSetAggregateChanged` is the per-row change feed: use it to mirror aggregate values to an external store (Redis, Kafka, Reverb, search index) without polling. The firing site short-circuits when no listener is registered, so the package pays no SELECT cost for users who don't want the feed; when at least one listener is attached, each maintenance pass issues one extra `SELECT id, lft, col…` over the targeted ancestor chain before and after the UPDATE to capture old / new values. Five stages: `on_create`, `on_update` (source-column changes), `on_delete`, `move`, `on_restore`. Internal companion columns (the `__sum` / `__count` auto-promotions behind AVG, Variance, WeightedAvg, etc.) are excluded — only user-declared aggregate columns produce events.

`AggregateDriftDetected` only fires when drift exists — pair it with a periodic monitoring job that calls `aggregateErrors()` to get an alert when something actually goes wrong.

### Scope guard

| Event                    | Fires when                                               | Payload                          |
| ------------------------ | -------------------------------------------------------- | -------------------------------- |
| `ScopeViolationDetected` | immediately before a `ScopeViolationException` is thrown | `modelClass`, `stage`, `message` |

Useful for security / audit signals: cross-scope writes on multi-tenant trees are almost always a permission boundary mistake. The exception still propagates — this event lets observability tooling distinguish the package's scope guard from generic exceptions.

## Recipes

### Bulk-index a freshly imported subtree

```php
use Vusys\NestedSet\Events\BulkInsert\BulkInsertTreeSaved;

Event::listen(BulkInsertTreeSaved::class, function (BulkInsertTreeSaved $e): void {
    if ($e->modelClass !== Category::class) {
        return;
    }

    // One round-trip to the search engine instead of N from the per-row
    // `created` listener.
    Search::indexMany($e->nodes);
});
```

### Cache-evict every descendant on cascade

The per-row Eloquent `deleted` event never fires for cascaded descendants — listen for the cascade event instead.

```php
use Vusys\NestedSet\Events\SoftDelete\SubtreeSoftDeleted;
use Vusys\NestedSet\Events\Subtree\SubtreeForceDeleted;

Event::listen(SubtreeSoftDeleted::class, function (SubtreeSoftDeleted $e): void {
    foreach ($e->descendantIds as $id) {
        Cache::forget("category:{$id}");
    }
});

Event::listen(SubtreeForceDeleted::class, function (SubtreeForceDeleted $e): void {
    foreach ($e->descendantIds as $id) {
        Cache::forget("category:{$id}");
        Search::removeById($id);
    }
});
```

### Invalidate breadcrumb cache after a subtree move

When an interior node moves, its descendants' ancestor chains have all changed. `SubtreeMoved` gives you the full list in one signal.

```php
use Vusys\NestedSet\Events\Subtree\SubtreeMoved;

Event::listen(SubtreeMoved::class, function (SubtreeMoved $e): void {
    Cache::forget("breadcrumbs:{$e->anchor->getKey()}");
    foreach ($e->descendantIds as $id) {
        Cache::forget("breadcrumbs:{$id}");
    }
});
```

### Audit log for cross-tenant scope violations

```php
use Vusys\NestedSet\Events\Diagnostics\ScopeViolationDetected;

Event::listen(ScopeViolationDetected::class, function (ScopeViolationDetected $e): void {
    Log::channel('audit')->warning('nestedset scope violation', [
        'model' => $e->modelClass,
        'stage' => $e->stage,   // 'mutation' | 'repair' | 'bulk_insert' | 'queue_dispatch'
        'detail' => $e->message,
        'user' => auth()->id(),
    ]);
});
```

### Streaming progress for chunked repairs

```php
use Vusys\NestedSet\Events\Aggregates\FixAggregatesChunkCompleted;

Event::listen(FixAggregatesChunkCompleted::class, function (FixAggregatesChunkCompleted $e): void {
    Log::info("nestedset chunk {$e->chunkIndex}: {$e->rowsUpdated} rows in {$e->durationMs}ms");
});
```

### Mirror aggregate columns to Redis / Kafka / Reverb

`NestedSetAggregateChanged` is a CDC-style change feed for maintained aggregate columns. Listen for it to mirror per-row aggregate values to any external store — no polling, no extra read-side query.

```php
use Vusys\NestedSet\Events\Aggregates\NestedSetAggregateChanged;

Event::listen(NestedSetAggregateChanged::class, function (NestedSetAggregateChanged $e): void {
    Redis::hset(
        "aggregates:{$e->modelClass}:{$e->nodeId}",
        $e->column,
        is_scalar($e->newValue) ? (string) $e->newValue : '',
    );
});
```

The event is **opt-in by listener presence** — without a registered listener, the firing site short-circuits before issuing the snapshot SELECTs, so the package's hot path stays at the same cost as before. As soon as any listener attaches, every maintenance pass pays the cost of one extra SELECT pre- and post-update over the ancestor chain (plus one event dispatch per (row, column) that actually moved).

For Kafka / Reverb broadcast, just hand the event object straight to your publisher — the payload is scalar and queue-safe:

```php
Event::listen(NestedSetAggregateChanged::class, function (NestedSetAggregateChanged $e): void {
    Kafka::publish('nestedset.aggregates', [
        'model' => $e->modelClass,
        'node' => $e->nodeId,
        'column' => $e->column,
        'old' => $e->oldValue,
        'new' => $e->newValue,
        'chain' => $e->ancestorChain,
        'stage' => $e->stage,
    ]);
});
```

`ancestorChain` is the same across every event emitted by a single mutation (it's the chain of node ids the maintenance pass touched, deepest first). Consumers that want to coalesce sibling events — e.g. roll up "five columns changed on these three rows" into one downstream message — can key on it.

### Sentry for hook failures

```php
use Vusys\NestedSet\Events\Aggregates\AggregateMaintenanceFailed;

Event::listen(AggregateMaintenanceFailed::class, function (AggregateMaintenanceFailed $e): void {
    Sentry::captureException($e->exception, [
        'tags' => [
            'nestedset_stage' => $e->stage,
            'nestedset_model' => $e->modelClass,
        ],
    ]);
});
```

### Datadog histogram for repair latency

```php
use Vusys\NestedSet\Events\Aggregates\FixAggregatesCompleted;

Event::listen(FixAggregatesCompleted::class, function (FixAggregatesCompleted $e): void {
    Datadog::histogram('nestedset.fix_aggregates.duration_ms', $e->durationMs, [
        'model' => $e->modelClass,
        'rows' => $e->totalRowsUpdated,
        'chunks' => $e->totalChunks,
    ]);
});
```

## Queue safety

Telemetry events are simple readonly value objects with scalar / array fields, so queued listeners (`ShouldQueue`) are safe.

Two categories need synchronous capture-and-forward if you want to queue work:

- **`AggregateMaintenanceFailed::$exception`** is a `Throwable` and won't serialise cleanly across most queue drivers.
- **Model-carrying events** hold live Eloquent instances and / or references to interface types that aren't safe to serialise without full Eloquent context: `BulkInsertTreeStarting`, `BulkInsertTreePlanned`, `BulkInsertNodeSaved`, `BulkInsertTreeSaved`, every `Subtree*` event, `NodesSwapped`, `SiblingsReordered`, `NodePromotedToRoot`, `SoftDeleteMarkerCaptured`.

The pattern is:

```php
use App\Jobs\IndexNodesJob;
use Vusys\NestedSet\Events\BulkInsert\BulkInsertTreeSaved;

Event::listen(BulkInsertTreeSaved::class, function (BulkInsertTreeSaved $e): void {
    // Capture queue-safe fields synchronously.
    $ids = array_map(fn ($n) => $n->getKey(), $e->nodes);
    $modelClass = $e->modelClass;

    // Forward those to a queued job that re-queries.
    IndexNodesJob::dispatch($modelClass, $ids);
});
```

## Ordering guarantees

Within a single Eloquent lifecycle hook (`saving` → `created` → `saved` or the deletion equivalents), package events fire in a deterministic order tied to the SQL the package issues. The contracts that matter:

- For an existing-node move: `SubtreeMoving` → structural SQL → `NodeMoved` → `SubtreeMoved` → (`NodePromotedToRoot` when applicable).
- For a soft delete: anchor's Eloquent `deleted` → `SubtreeSoftDeleting` → cascade UPDATE → `SubtreeSoftDeleted` → aggregate maintenance → `NodeAggregatesRecomputed`.
- For a restore: anchor's Eloquent `restoring` → `SoftDeleteMarkerCaptured` (the package buffers the `deleted_at` value before Eloquent clears it) → anchor's Eloquent `restored` → `SubtreeRestoring` → cascade UPDATE (nulls `deleted_at` on every descendant whose stamp matches the marker) → `SubtreeRestored` → aggregate maintenance → `NodeAggregatesRecomputed`.
- For a force delete on an interior node: anchor's Eloquent `deleting` → `SubtreeForceDeleting` → cascade DELETE → `SubtreeForceDeleted` → anchor's row delete → aggregate maintenance → `NodeAggregatesRecomputed`.
- For a bulk insert: `BulkInsertTreeStarting` → `BulkInsertTreePlanned` → (N × `creating` / `saving` / `created` / `saved` / `BulkInsertNodeSaved`) → closing `fixAggregates` (which emits `FixAggregatesCompleted` and `DeferredAggregateMaintenanceCompleted`) → `BulkInsertTreeSaved` → `BulkInsertTreeCompleted`.

The package does NOT guarantee ordering *between* the package's events and your own model observers — Eloquent runs observers in registration order. If you need a specific ordering, register your observers explicitly via `Model::observe()` after the package's service provider has booted.
