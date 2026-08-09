# Concurrency & Transactions

A nested-set mutation touches many rows, and the bounds sequence is only valid as a whole. Two writers racing on the same parent, or a half-applied move, would corrupt the tree. This page covers the three mechanisms that keep mutations atomic and serialised: the auto-transaction wrap, row-level locks, and the `aggregate_locking` knob. It is the implementation detail behind the user-facing [Transactions](../reference/transactions.html) page.

## Auto-transactions

Every mutation is wrapped in a database transaction by the trait's `save()` override (`src/Concerns/HasTreeMutation.php`):

```php
public function save(array $options = []): bool
{
    if (! config('nestedset.auto_transaction', true)) {
        return parent::save($options);
    }

    try {
        return (bool) $this->getConnection()->transaction(function () use ($options): bool {
            $saved = parent::save($options);

            // A saving/creating/updating listener cancelled the save by
            // returning false — but the trait's own `saving` listener has
            // already run the structural SQL (makeGap / moveNode). A bare
            // `return false` would commit that gap/move with no row write.
            // Throw to force the rollback; the catch restores the
            // cancelled-save contract for the caller.
            if ($saved === false) {
                throw new SaveCancelledException;
            }

            return $saved;
        });
    } catch (SaveCancelledException) {
        return false;
    }
}
```

This is what makes a mutation all-or-nothing. A single `save()` performs several distinct writes — the gap shift or `moveNode` (from the `saving` hook), the Eloquent INSERT/UPDATE itself, and the aggregate maintenance (from the `saved` / `created` hooks). Without the wrap, a failure partway through — a unique constraint, a throwing listener — would leave the gap committed and the row never inserted, a permanent hole in the `lft`/`rgt` sequence. The `SaveCancelledException` dance is load-bearing: because the structural SQL already ran in the `saving` listener, a listener that *cancels* the save (returns `false`) must still roll back, so the override converts the cancellation into a thrown exception inside the transaction and then re-presents it to the caller as a plain `false`. Laravel handles nested `transaction()` calls via savepoints, so wrapping inside your own outer transaction is safe.

Set `auto_transaction => false` only if you are managing transactions yourself at the call site — then *you* own the atomicity guarantee.

## Row locks

Transactions make a mutation atomic, but they don't by themselves stop two concurrent appenders from reading the same `parent.rgt` and both inserting at that slot — a `duplicate_lft` corruption. The engine prevents that with `SELECT … FOR UPDATE` row locks at the read that decides the insert position.

`appendToNode` / `prependToNode` / `insertBefore` / `insertAfter` all re-read the target's bounds with `lockForUpdate: true`:

```php
private function actAppendTo(Model&HasNestedSet $parent): void
{
    $parentBounds = $this->freshBoundsOf($parent, lockForUpdate: true);
    $position = $parentBounds->rgt;
    // ...
    $this->positionAt($position, $newDepth, $newParentId);
}
```

The lock is held for the rest of the enclosing transaction, so the gap-shift `UPDATE` that follows runs while a second appender is still blocked on the same `SELECT`. The second writer only proceeds once the first commits — and then it reads the *post-gap* `rgt`, landing in a fresh slot. The flag flows down to `getPlainNodeData()` in the mutation builder, which only applies the lock on backends that support it:

```php
if ($lockForUpdate && $this->connection->getDriverName() !== 'sqlite') {
    $query->lockForUpdate();
}
```

### Lock ordering and deadlocks

A move locks both the **mover**'s bounds and the **target**'s bounds. The two locks are taken in the order the operation reaches them, which is **not** canonicalised across operations. Two *crossing* moves running concurrently — say, "move A under B" and "move B under A" — therefore acquire the same two rows in opposite orders and can deadlock. This is safe for data integrity: the engine detects the cycle and aborts one transaction (SQLSTATE `40001` / `40P01` on the supported backends), rolling its half-applied gap-shift back cleanly. It is *not* transparent to the caller — the aborted side surfaces a `QueryException`.

Treat a move (or any contended tree write) the way you'd treat any other transaction that can deadlock: **retry on the serialization-failure SQLSTATEs.** The package's own fork-based concurrency tests do exactly this (`withDeadlockRetry()` in the test suite). A small bounded-retry wrapper around the write is the standard remedy; the package does not retry for you because the right backoff/attempt policy is application-specific.

### `makeRoot()` and the PostgreSQL aggregate-lock quirk

Creating a root reads `max(rgt)` in the scope, which has its own concurrency hazard — two parallel `makeRoot()` calls could read the same max and collide. Locking that read is backend-sensitive, and `actMakeRoot()` handles three cases:

```php
if ($driver === 'sqlite') {
    $rawMax = $query->max($this->getRgtName());
} else {
    $rawMax = $query
        ->orderBy($this->getRgtName(), 'desc')
        ->limit(1)
        ->lockForUpdate()
        ->value($this->getRgtName());
}
```

- **SQLite** is single-writer, so no lock is needed.
- **PostgreSQL** rejects `FOR UPDATE` on an aggregate query (`SELECT max(...) FOR UPDATE` → SQLSTATE `0A000`). So instead of locking the aggregate, the code locks the single row that *holds* the max via `ORDER BY rgt DESC LIMIT 1 FOR UPDATE` — same value, but a lockable row. This also works on MySQL/MariaDB.

The scope filter on the max lookup matters too: without it, the second scope's first root would land past the first scope's `rgt` and silently break per-scope bound independence (the comment notes this is caught by the scope-isolation fuzzer).

### Backend differences in one place

| Backend         | Row locking                                 | Notes                                                                  |
| --------------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| PostgreSQL      | `FOR UPDATE` on rows; **not** on aggregates | `makeRoot` locks the max-`rgt` row, not `max()`                        |
| MySQL / MariaDB | `FOR UPDATE`                                | `SET` evaluates left-to-right (why `depth` is set first in `moveNode`) |
| SQLite          | single-writer; locks are no-ops             | the `lockForUpdate` flag short-circuits                                |

## Aggregate locking — the recompute race {#aggregate-locking}

Delta-maintained aggregates (SUM/COUNT) update ancestors with a single self-relative `UPDATE` (`col = col ± Δ`), which is safe under the engine's ordinary row locks. But the **recompute** path (MIN/MAX, raw filters, `fixAggregates`) is two statements — a `SELECT` of the recomputed value followed by an `UPDATE` — and two writers recomputing overlapping subtrees concurrently can interleave and drift. The `aggregate_locking` config flag (`config/nestedset.php`) controls whether the recompute `SELECT` takes a `FOR UPDATE` lock on the ancestor chain first:

| Value            | Behaviour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `auto` (default) | Lock the ancestor chain only on the recompute path (MIN/MAX, `fixAggregates`). Delta-only updates rely on the engine's single-statement row locks — sufficient under default isolation on every supported backend. The right setting for nearly every app. **PG `READ COMMITTED` caveat:** the recompute `SELECT` computes the value and takes its `FOR UPDATE` lock in one statement, so the locked outer rows are re-fetched but the correlated descendant subqueries still read the statement snapshot — a descendant change committing in that window can leave a recomputed MIN/MAX value briefly stale until the next pass (`fixAggregates($anchor)` reconciles it). |
| `always`         | Lock the ancestor chain before *every* aggregate `UPDATE`, deltas included. Choose this under non-default isolation (e.g. PostgreSQL `REPEATABLE READ`) or if you've observed drift under concurrent load.                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| `never`          | No explicit locks. Marginally faster on the recompute path; can drift on PostgreSQL `READ COMMITTED` with concurrent recomputes against overlapping subtrees.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

The strategy split that this flag governs is described in [Aggregate Maintenance](aggregate-maintenance.html#the-four-families). The key takeaway: a *delta* is commutative and self-correcting under row locks; a *recompute* reads-then-writes and needs the chain held still in between.

## Event dispatch — observable but free when unused {#event-dispatch}

Every meaningful operation fires a typed telemetry event, but the dispatch is designed to cost nothing when nobody is listening. `EventDispatcher` (`src/Events/EventDispatcher.php`) is a static funnel gated on config:

```php
public static function dispatch(object $event): void
{
    if (! self::enabled()) {
        return;
    }

    event($event);
}
```

The more important gate is `hasListeners()`, used by firing sites that would otherwise do real work — like an extra `SELECT` to gather descendant ids for a cascade event — *before* dispatching:

```php
public static function hasListeners(string $event): bool
{
    if (! self::enabled()) {
        return false;
    }

    return resolve(Dispatcher::class)->hasListeners($event);
}
```

You can see the pattern at the call sites: `SubtreeMoved`'s descendant-id list is only collected when a listener is registered —

```php
$descendantIds = EventDispatcher::hasListeners(SubtreeMoved::class)
    ? $this->collectStrictDescendantIds($toBounds)
    : [];
```

— so an unobserved move pays no extra query. Setting `events_enabled => false` short-circuits the entire surface in one line. The full event catalogue, payloads, and ordering guarantees live on the [Events reference](../reference/events.html) page; this is just the dispatch *mechanism* behind it.

## Where to go next

This is the last page of the Internals section. To see how the structural and aggregate writes are sequenced within a single transaction, revisit [Aggregate Maintenance → hook ordering](aggregate-maintenance.html#lifecycle-hook-ordering).
