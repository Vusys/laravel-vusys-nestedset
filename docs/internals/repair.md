# Integrity & Repair

The `lft`/`rgt`/`depth` index can drift — a crashed transaction, an out-of-band `UPDATE`, a `parent_id` change made without the mutation API. Because [`parent_id` is the source of truth](nested-set-model.html#parent-id-source-of-truth), the package can always detect the drift and rebuild the index from scratch. This page walks the detection queries and the rebuild algorithm.

Two files: `src/Concerns/HasTreeRepair.php` (the static API and scope guards) and `src/Query/TreeRepairBuilder.php` (the detection + rebuild engine). This is the implementation behind the user-facing [Tree Repair](../maintenance/fix-tree.html) and [Corruption Reference](../maintenance/corruption.html) pages.

## Detecting corruption — `countErrors()`

`countErrors()` runs four scoped queries, one per corruption category (`src/Query/TreeRepairBuilder.php`):

```php
public function countErrors(): array
{
    $invalidBounds = (int) $this->scoped()
        ->whereColumn($this->lft, '>=', $this->rgt)
        ->count();

    $duplicateLft = (int) $this->scoped()
        ->select($this->lft)
        ->groupBy(array_merge(array_keys($this->scope), [$this->lft]))
        ->havingRaw('COUNT(*) > 1')
        ->count();

    $duplicateRgt = (int) $this->scoped()
        ->select($this->rgt)
        ->groupBy(array_merge(array_keys($this->scope), [$this->rgt]))
        ->havingRaw('COUNT(*) > 1')
        ->count();

    $orphans = (int) $this->orphanQuery()->count();

    return [
        'invalid_bounds' => $invalidBounds,
        'duplicate_lft'  => $duplicateLft,
        'duplicate_rgt'  => $duplicateRgt,
        'orphans'        => $orphans,
    ];
}
```

| Category         | Detection                                                | Maps to invariant                |
| ---------------- | -------------------------------------------------------- | -------------------------------- |
| `invalid_bounds` | `lft >= rgt`                                             | `lft < rgt` for every node       |
| `duplicate_lft`  | `GROUP BY (scope…, lft) HAVING COUNT(*) > 1`             | `lft` unique within scope        |
| `duplicate_rgt`  | `GROUP BY (scope…, rgt) HAVING COUNT(*) > 1`             | `rgt` unique within scope        |
| `orphans`        | self-LEFT-JOIN where `parent_id` points at a missing row | every non-root has a real parent |

The duplicate checks group on `(scope columns + bound)` so a value that repeats across two different trees in a multi-tree table is *not* flagged — each scope has its own independent `1..2N` sequence. The orphan query is the subtle one: it LEFT-JOINs the table to itself on `parent.id = child.parent_id` **and** equates every scope column across the two sides:

```php
->leftJoin("{$tableName} as parent", function ($join) use ($scopeColumns): void {
    $join->on("parent.{$this->idCol}", '=', "child.{$this->parentId}");
    foreach ($scopeColumns as $column) {
        $join->on("parent.{$column}", '=', "child.{$column}");
    }
})
->whereNotNull("child.{$this->parentId}")
->whereNull("parent.{$this->idCol}");
```

Without the scope equality in the JOIN, a child whose `parent_id` happens to match a row in a *different* scope would join successfully and mask the orphan.

`isBroken()` is just `array_sum(countErrors()) > 0`. Both fire a `TreeIntegrityChecked` telemetry event on every call — including clean trees, so it doubles as a monitoring heartbeat ([Events](../reference/events.html#tree-repair)).

## Rebuilding — from `parent_id` to fresh bounds

`fixTree()` reconstructs `lft`/`rgt`/`depth` purely from the `parent_id` forest. `rebuildTree()` (whole scope) reads the id→parent map, partitions it into roots and a `children` adjacency list, walks it to assign coordinates, and bulk-writes the result inside a transaction:

```php
public function rebuildTree(): void
{
    $rows = $this->scoped()
        ->select([$this->idCol, $this->parentId])
        ->get()
        ->keyBy($this->idCol);

    $children = [];
    $roots = [];

    foreach ($rows as $id => $row) {
        $pid = $row->{$this->parentId};
        if ($pid === null) {
            $roots[] = $id;
        } else {
            $children[$pid][] = $id;
        }
    }

    $positions = $this->walkAssignPositions($roots, $children, startLft: 1, startDepth: 0);

    $this->connection->transaction(function () use ($positions): void {
        $this->bulkWritePositions($positions);
    });
}
```

### The walk is iterative, not recursive

`walkAssignPositions()` is a pre-order DFS that stamps `lft` on the way *in* and `rgt` on the way *out* — exactly the numbering described in [The Nested-Set Model](nested-set-model.html#the-encoding-by-example). It is written as an explicit stack of `enter`/`exit` tasks rather than a recursive function:

```php
private function walkAssignPositions(array $roots, array $children, int $startLft, int $startDepth): array
{
    $positions = [];
    $counter = $startLft;

    $tasks = [];
    foreach (array_reverse($roots) as $rootId) {
        $tasks[] = ['type' => 'enter', 'id' => $rootId, 'depth' => $startDepth];
    }

    while ($tasks !== []) {
        $task = array_pop($tasks);

        if ($task['type'] === 'exit') {
            $entry = $positions[$task['id']];
            $entry['rgt'] = $counter++;          // stamp rgt after all descendants
            $positions[$task['id']] = $entry;
            continue;
        }

        $positions[$task['id']] = ['lft' => $counter++, 'rgt' => 0, 'depth' => $task['depth']];

        // Queue exit before children so it pops after every descendant.
        $tasks[] = ['type' => 'exit', 'id' => $task['id']];

        // Children reversed so the first child pops first (stable pre-order).
        foreach (array_reverse($children[$task['id']] ?? []) as $childId) {
            $tasks[] = ['type' => 'enter', 'id' => $childId, 'depth' => $task['depth'] + 1];
        }
    }

    return $positions;
}
```

> [!IMPORTANT]
> The iterative shape is deliberate, not stylistic. The source comment notes that recursion bottomed out around xdebug's 256-frame default and PHP's \~10K-frame ceiling — both reachable on a "tall and skinny" corrupted tree, which is exactly the shape `fixTree()` exists to repair. Pushing the traversal onto a heap-allocated `$tasks` stack removes the limit. The pushing order (exit before children, children reversed) reproduces the same visit order a recursive walk would produce.

### Bulk-writing the positions

A naive rebuild would issue one `UPDATE` per row — 10K rows, 10K round-trips, multi-second wall-clock. `bulkWritePositions()` instead chunks the ids and emits one `UPDATE … SET col = CASE id WHEN … END` per chunk, with three `CASE` expressions (lft/rgt/depth) per statement:

```php
$sql = "UPDATE {$this->table} "
    ."SET {$this->lft} = ({$lftCase}), "
    ."{$this->rgt} = ({$rgtCase}), "
    ."{$this->depth} = ({$depthCase}) "
    ."WHERE {$this->idCol} IN ({$idPlaceholders}){$scopeClause}";
```

At the default `chunkSize = 500`, a 10K-row rebuild becomes \~20 `UPDATE`s instead of 10K. (This is the same chunked-CASE pattern the aggregate-repair path uses — see `AggregateDiffer`.) Scope predicates ride along in the `WHERE` so the rebuild can never escape the partition the builder was constructed for.

### Subtree rebuilds and the size delta

`rebuildSubtree($rootId)` repairs one subtree without touching its siblings — the path taken when you pass an anchor. It collects the subtree's ids by walking `parent_id` (`collectSubtree()`, an iterative BFS for the same stack-safety reason), then re-numbers starting from the root's existing `lft`. The wrinkle: if descendants were added or removed via `parent_id` without matching gap operations, the subtree's new slot count won't fit the band already reserved between the root's `lft` and `rgt`. So it shifts the surrounding rows by the delta first:

```php
$newSize = count($inSubtree) * 2;
$reservedSize = $reservedRgt - $startLft + 1;
$delta = ($rootRow !== null && $reservedSize >= 2) ? $newSize - $reservedSize : 0;

// ... inside the transaction:
if ($delta > 0) {
    $mutator->makeGap($reservedRgt + 1, $delta);   // need more room
} else {
    $mutator->closeGap($reservedRgt, -$delta);     // reclaim space
}
$this->bulkWritePositions($positions);
```

Without that shift, the rebuilt subtree's tail would collide with whichever sibling sits at `rgt + 1`. It reuses the same `makeGap`/`closeGap` primitives the [mutation engine](mutation-engine.html#opening-and-closing-gaps) uses.

## Structural repair then aggregate repair

The model-facing `fixTree()` in `HasTreeRepair` chains the two repair passes so a single call leaves both the structure *and* the stored rollups correct:

```php
$treeResult = $builder->fixTree($rootId);
$aggregatesFixed = self::runFixAggregates($anchor, $rootId);
```

The order matters — aggregates are recomputed against the *post-repair* structure. The combined result is returned as a `TreeFixResult` (carrying the node count, post-repair error counts, and an optional `AggregateFixResult`), and a `FixTreeCompleted` event is fired with the wall-clock duration of both passes. The aggregate side is covered in [Aggregate Maintenance](aggregate-maintenance.html#repair-fixaggregates).

## The scope guard

`repairBuilder()` enforces a safety rule for multi-tree models: if the class declares a scope and no anchor is passed, it dispatches a `ScopeViolationDetected` event and throws `ScopeViolationException`:

```php
if ($scopeColumns !== [] && ! $anchor instanceof HasNestedSet) {
    // ... dispatch ScopeViolationDetected ...
    throw new ScopeViolationException($message);
}
```

The rationale is in the docblock: an unscoped `fixTree()` on a scoped table would walk every tree in a potentially multi-million-row table — almost never what the caller intended. `fixTree()` additionally rejects an anchor whose primary key is null (an unsaved model), because a null PK silently collapses to "no root id" and widens the rebuild to the whole scope. Read paths (`isBroken`/`countErrors`) stay permissive, since they're sometimes called with a stub anchor used purely as a scope carrier.

## Where to go next

[Concurrency & Transactions](concurrency.html) covers how all of these operations stay correct under concurrent writers.
