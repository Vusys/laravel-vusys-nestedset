# The Mutation Engine

This is the heart of the package: how an `appendToNode()` or a subtree move turns into SQL that renumbers `lft`/`rgt` without ever leaving the tree in a broken state. Two files do the work — `src/Concerns/HasTreeMutation.php` (the API and orchestration) and `src/Query/TreeMutationBuilder.php` (the atomic SQL).

## Mutations are queued, then dispatched

The public methods do almost nothing. `appendToNode()` just records what you want and returns `$this` for chaining:

```php
public function appendToNode(HasNestedSet $parent): static
{
    $this->pending = new PendingOperation('appendTo', $parent);

    return $this;
}
```

`PendingOperation` (`src/PendingOperation.php`) is a three-field value object — the action name, the target neighbour (parent or sibling), and a `Position` enum (`Before`/`After`) for sibling inserts:

```php
readonly class PendingOperation
{
    public function __construct(
        public string $action,
        public ?HasNestedSet $node = null,
        public Position $position = Position::After,
    ) {}
}
```

`makeRoot`, `prependToNode`, `insertBeforeNode`, and `insertAfterNode` each set a different `PendingOperation`. The ergonomic helpers fold into these: `moveTo($parent, $position)` resolves a string/index position to one of append/prepend/insertBefore; `moveBefore`/`moveAfter` alias the sibling inserts; `up()`/`down()` swap with an adjacent sibling.

Nothing has touched the database yet. The actual write happens on `save()`, which the trait overrides to dispatch the pending action from the `saving` event (wired in `bootNodeTrait()` — see [Architecture](architecture.html#lifecycle-wiring)). `callPendingAction()` is the dispatcher:

```php
public function callPendingAction(): void
{
    if ($this->pending === null) {
        return;
    }

    $op = $this->pending;
    $this->pending = null;

    if ($op->node !== null) {
        $target = $this->requireModelNode($op);
        NestedSetScopeResolver::assertSameScope($this, $target);
    }

    // ... capture pre-move bounds, dispatch events ...

    match ($op->action) {
        'appendTo'  => $this->actAppendTo($this->requireModelNode($op)),
        'prependTo' => $this->actPrependTo($this->requireModelNode($op)),
        'sibling'   => $this->actSibling($this->requireModelNode($op), $op->position),
        'root'      => $this->actMakeRoot(),
        default     => throw new LogicException("Unknown pending action: {$op->action}"),
    };
}
```

Deferring the write to `save()` is what lets the gap-shift, the Eloquent INSERT/UPDATE, and the aggregate hooks all share one transaction ([Concurrency & Transactions](concurrency.html)).

## Resolving a target position

Each `act*` method translates the operation into a single `(position, depth, parent_id)` triple, then hands off to `positionAt()`. They all re-read the target's bounds from the database first — the in-memory copy you passed in may be stale:

```php
private function actAppendTo(Model&HasNestedSet $parent): void
{
    $parentBounds = $this->freshBoundsOf($parent, lockForUpdate: true);
    $position    = $parentBounds->rgt;          // just before parent's closing boundary
    $newDepth    = $parentBounds->depth + 1;
    $newParentId = $this->keyOf($parent);

    $this->positionAt($position, $newDepth, $newParentId);
}
```

The position each operation targets:

| Operation                | `position` (a target `lft`) | Depth              |
| ------------------------ | --------------------------- | ------------------ |
| `appendTo($parent)`      | `parent.rgt`                | `parent.depth + 1` |
| `prependTo($parent)`     | `parent.lft + 1`            | `parent.depth + 1` |
| `insertBefore($sibling)` | `sibling.lft`               | `sibling.depth`    |
| `insertAfter($sibling)`  | `sibling.rgt + 1`           | `sibling.depth`    |
| `makeRoot()`             | `max(rgt) + 1` in scope     | `0`                |

The `lockForUpdate: true` on the parent/sibling read is the concurrency guard — covered in [Concurrency & Transactions](concurrency.html#row-locks).

## `positionAt()` — new node vs. existing node

`positionAt()` is where the two write paths fork. A brand-new node only needs a gap opened for it; an existing node needs its whole subtree moved.

```php
private function positionAt(int $position, int $newDepth, int|string|null $newParentId): void
{
    $mutator = $this->newTreeMutator();

    if (! $this->exists) {
        $mutator->makeGap($position, 2);

        $this->setAttribute($this->getLftName(), $position);
        $this->setAttribute($this->getRgtName(), $position + 1);
        $this->setAttribute($this->getDepthName(), $newDepth);
        $this->setAttribute($this->getParentIdName(), $newParentId);

        return;
    }

    // Existing node: read fresh bounds, move the subtree, re-read result.
    $from = $mutator->getNodeData($this->keyOf($this));
    $depthDelta = $newDepth - $from->depth;

    $mutator->moveNode($from, $position, $depthDelta);

    $newBounds = $mutator->getPlainNodeData($this->keyOf($this));
    $this->setAttribute($this->getLftName(), $newBounds['lft']);
    $this->setAttribute($this->getRgtName(), $newBounds['rgt']);
    $this->setAttribute($this->getDepthName(), $newBounds['depth']);
    $this->setAttribute($this->getParentIdName(), $newParentId);

    $this->syncOriginalAttribute($this->getLftName());
    $this->syncOriginalAttribute($this->getRgtName());
    $this->syncOriginalAttribute($this->getDepthName());
}
```

For a **new** node: open a 2-wide gap and drop the node into `[position, position + 1]`. The Eloquent INSERT that follows `save()` writes those attributes. For an **existing** node: `moveNode()` shifts many rows in one SQL statement, so the in-memory model can't predict its own new bounds — the code re-reads them with `getPlainNodeData()` and calls `syncOriginalAttribute()` so Eloquent's dirty-tracking doesn't try to write `lft`/`rgt` a second time.

## Opening and closing gaps

`makeGap()` and `closeGap()` (`src/Query/TreeMutationBuilder.php`) are the simplest mutations — they slide a contiguous block of the number line to make or reclaim room. Each is a single `UPDATE` with a `CASE WHEN` per column:

```php
public function makeGap(int $at, int $size): void
{
    $this->scoped()->update([
        $this->lft => new TreeExpression(
            "CASE WHEN {$this->lft} >= {$at} THEN {$this->lft} + {$size} ELSE {$this->lft} END"
        ),
        $this->rgt => new TreeExpression(
            "CASE WHEN {$this->rgt} >= {$at} THEN {$this->rgt} + {$size} ELSE {$this->rgt} END"
        ),
    ]);
}
```

`makeGap($at, $size)` pushes every `lft`/`rgt` value `>= $at` up by `$size`, opening a `$size`-wide hole. `closeGap($at, $size)` is the mirror image — everything `> $at` slides down by `$size` — and runs after a hard delete to keep the bounds a contiguous `1..2N` permutation.

A worked insert: to add a new leaf as the last child of a node whose `rgt` is `7`, `makeGap(7, 2)` shifts the `7` boundary and everything after it up by 2, freeing slots `7` and `8`; the new leaf takes `[7, 8]`:

```text
before:  … 5 6   7(parent.rgt) …          values >= 7 shift +2
after:   … 5 6   [7 8]new   9(parent.rgt) …
```

`TreeExpression` (`src/Query/TreeExpression.php`) is what lets these expressions reach the database verbatim — it wraps the raw SQL string as an `Expression` so the query builder splices it into the `SET` clause rather than binding it as a literal.

## Moving a subtree — the `CASE WHEN` trick

Moving an existing subtree is the most intricate operation, because the moved rows *and* the rows they pass over both have to shift, in opposite directions, and the result must be valid the instant the statement commits. `moveNode()` does it in one `UPDATE`:

```php
public function moveNode(NodeBounds $from, int $position, int $depthDelta): void
{
    $lft = $from->lft;
    $rgt = $from->rgt;
    $height = $rgt - $lft + 1;

    if ($lft < $position && $position <= $rgt) {
        throw new \LogicException('Cannot move node into itself.');
    }

    $boundFrom = min($lft, $position);
    $boundTo   = max($rgt, $position - 1);

    $distance = $boundTo - $boundFrom + 1 - $height;

    if ($distance === 0 && $depthDelta === 0) {
        return;
    }

    if ($position > $lft) {
        $subtreeShift   = $distance;   // forward
        $bystanderShift = -$height;
    } else {
        $subtreeShift   = -$distance;  // backward
        $bystanderShift = $height;
    }

    $this->scoped()->update([
        $this->depth => new TreeExpression(
            "CASE WHEN {$this->lft} BETWEEN {$lft} AND {$rgt} "
            ."THEN {$this->depth} + {$depthDelta} ELSE {$this->depth} END"
        ),
        $this->lft => new TreeExpression(
            $this->shiftCase($this->lft, $lft, $rgt, $boundFrom, $boundTo, $subtreeShift, $bystanderShift),
        ),
        $this->rgt => new TreeExpression(
            $this->shiftCase($this->rgt, $lft, $rgt, $boundFrom, $boundTo, $subtreeShift, $bystanderShift),
        ),
    ]);
}
```

### The band

The idea (credited in the source to the kalnoy/nestedset move trick): only rows inside the **band** `[boundFrom, boundTo]` move at all — everything outside it is untouched. Within the band there are two groups:

- the **moving subtree** (`lft..rgt`), which shifts by `subtreeShift`;
- the **bystanders** — rows the subtree slides past — which shift by `bystanderShift` to fill the space the subtree vacates (or to open the space it needs). The bystander shift is exactly `∓height`, because the subtree occupies `height` slots.

### `WHEN`-clause ordering

`shiftCase()` emits the per-column `CASE`, and the ordering of the `WHEN`s is load-bearing:

```php
return 'CASE '
    ."WHEN {$col} BETWEEN {$lft} AND {$rgt} THEN {$col} {$subtree} "
    ."WHEN {$col} BETWEEN {$boundFrom} AND {$boundTo} THEN {$col} {$bystander} "
    ."ELSE {$col} END";
```

The subtree band (`lft..rgt`) is *inside* the bystander band (`boundFrom.. boundTo`), so the subtree `WHEN` must come first — a moving row matches it and never reaches the bystander clause. `depth` is listed first in the `SET` array because MySQL evaluates `SET` left-to-right against pre-update values, and the depth `CASE` keys off the still-original `lft`.

### Worked example

Start from this tree (numbers are `lft..rgt`, `dN` is depth):

```text
Root  1..12  d0
├── A   2..3   d1
├── B   4..9   d1
│   ├── B1  5..6  d2
│   └── B2  7..8  d2
└── C  10..11  d1
```

Now `A->appendToNode(B)->save()` — move `A` to be the last child of `B`. For an append, `position = B.rgt = 9`, and `A`'s new depth is `B.depth + 1 = 2`, so `depthDelta = +1`. Plugging into `moveNode(from = A{2,3}, position = 9, depthDelta = 1)`:

- `height = 3 - 2 + 1 = 2`
- `boundFrom = min(2, 9) = 2`, `boundTo = max(3, 8) = 8` → band is `2..8`
- `distance = 8 - 2 + 1 - 2 = 5`
- `position(9) > lft(2)`, so `subtreeShift = +5`, `bystanderShift = -2`

Applying the `CASE`s to every row:

| Node | old `lft..rgt` | clause                           | new `lft..rgt` | new depth |
| ---- | -------------- | -------------------------------- | -------------- | --------- |
| Root | 1..12          | ELSE / ELSE                      | 1..12          | 0         |
| A    | 2..3           | subtree `+5`                     | **7..8**       | 1→**2**   |
| B    | 4..9           | `lft` bystander `-2`, `rgt` ELSE | **2..9**       | 1         |
| B1   | 5..6           | bystander `-2`                   | **3..4**       | 2         |
| B2   | 7..8           | bystander `-2`                   | **5..6**       | 2         |
| C    | 10..11         | ELSE / ELSE                      | 10..11         | 1         |

(`B.rgt = 9` falls outside the `2..8` band, so it stays at 9 — the subtree moved *into* B, widening it.) The result is a valid tree, reached in one statement:

```text
Root  1..12  d0
├── B   2..9   d1
│   ├── B1  3..4  d2
│   ├── B2  5..6  d2
│   └── A   7..8  d2   ← now last child of B
└── C  10..11  d1
```

`positionAt()` then re-reads `A`'s new bounds (`7..8`, depth `2`) and the subsequent Eloquent UPDATE writes only the changed `parent_id`. The `distance === 0 && depthDelta === 0` early return makes a no-op move (e.g. re-appending a node that is already the last child) cost nothing.

## `makeRoot()` and the per-scope max

`actMakeRoot()` positions the node at `max(rgt) + 1` within its scope, depth 0, `parent_id` null — appending a fresh tree at the tail of the number line. Reading the max is backend-sensitive: PostgreSQL rejects `FOR UPDATE` on an aggregate query, so instead of `lockForUpdate()->max()` the code locks the single highest-`rgt` row via `ORDER BY rgt DESC LIMIT 1 FOR UPDATE` (and skips the lock entirely on single-writer SQLite). The reasoning is in [Concurrency & Transactions](concurrency.html#row-locks).

## The aggregate seams

For existing-node moves, `callPendingAction()` brackets the structural SQL with two seams so aggregate maintenance can act on the correct ancestor chain:

```php
protected function onBeforePendingAction(NodeBounds $from, string $action): void
{
    $this->applyAggregateBeforeMove($from, $action);   // subtract from OLD ancestors
}

protected function onAfterPendingAction(NodeBounds $from, NodeBounds $to, string $action): void
{
    $this->applyAggregateAfterMove($from, $to, $action); // add to NEW ancestors
}
```

The "before" hook runs while the pre-move bounds are still accurate; the "after" hook runs once the rows are renumbered. Both are inside the same transaction as the move. How they recompute the rollups is the subject of [Aggregate Maintenance](aggregate-maintenance.html#moves).

## Deletes and gap closure

Deletes are handled in the `deleted` lifecycle hook, not the mutation methods. For a hard delete of an interior node, `applyForceDeleteCascade()` first removes every descendant in the same scope with a single bounds-scoped `DELETE`, then `applyStructuralCleanupOnDelete()` calls `closeGap(lft, rgt - lft + 1)` to reclaim the whole subtree's slots. Soft deletes skip the gap close — the rows still exist and must keep their slots. The cascade and ordering are detailed in [Aggregate Maintenance → Soft deletes](aggregate-maintenance.html#soft-deletes).

## Where to go next

[Query Engine & Relations](query-engine.html) covers the read side — how the same `lft`/`rgt` bounds power `BETWEEN`-based descendant and ancestor queries.
