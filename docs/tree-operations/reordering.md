# Reordering Siblings

`reorderChildren()` reshuffles a parent's direct children into a new order using **one** atomic `CASE WHEN UPDATE`. Use it for drag-and-drop UIs and "sort siblings by X" — anywhere the user supplies the final order as a list and the server is expected to make it so.

> **Reordering vs moving.** `reorderChildren()` is for *same-parent* reorders. To move a node to a different parent, use `moveTo($parent, $position)` (see [Inserting & Moving](inserting.html)). The two primitives are deliberately separate: a same-parent reshuffle is one UPDATE bounded by the parent's subtree window, while a cross-parent move shifts every row in the affected ancestor chains. Mixing them in one method would hide that cost asymmetry from the caller.

## reorderChildren — apply a flat order

Pass the parent's direct-child primary keys (or model instances) in the desired order:

```php
$electronics->reorderChildren([
    $laptops->id,
    $tablets->id,
    $phones->id,
]);
```

### What the reorder actually does

Look at the `lft` / `rgt` pill badges on each row. Before the call, the children are ordered `phones, tablets, laptops`:

```ns-tree
Electronics
  Phones
    iPhone
    Android
  Tablets
  Laptops
    MacBook
    ThinkPad
```

After the call, the same rows carry the same `name`, `depth`, and `parent_id` — but their `lft` / `rgt` slots have been swapped to match the requested order. Each subtree slides as a block; `iPhone` stays under `Phones`, `MacBook` stays under `Laptops`:

```ns-tree
Electronics
  Laptops
    MacBook
    ThinkPad
  Phones
    iPhone
    Android
  Tablets
```

This is **one** `CASE WHEN UPDATE`, bounded by `Electronics`'s `lft` / `rgt` window — one round-trip, no ancestor-chain work, no aggregate maintenance (`SUM(cost)` etc. stays correct because no subtree composition changed). Compare to `moveTo($differentParent, ...)`, which would shift every row in *two* ancestor chains; that asymmetry is why reorder and move are separate primitives.

After the call, `Category::where('parent_id', $electronics->id)->orderBy('lft')->pluck('id')` matches `[laptops, tablets, phones]`. Each sibling's descendants shift with it — `depth` and `parent_id` are unchanged for every row.

The membership must match exactly: every direct child appears once, no unknown keys, no duplicates. Any deviation throws `InvalidSiblingOrderException` with a message naming the offending keys.

| Condition                                    | Behaviour                                |
| -------------------------------------------- | ---------------------------------------- |
| Identity order (already matches `lft` order) | Silent no-op — no UPDATE fires, no event |
| Parent has no children                       | Silent no-op                             |
| Missing or extra key                         | `InvalidSiblingOrderException`           |
| Duplicate key                                | `InvalidSiblingOrderException`           |
| Parent unsaved                               | `UnplacedNodeException`                  |

## moveToSiblingPosition — drop a row at slot N

When you already have a child reference and want it at a specific 0-indexed position within its current sibling group:

```php
$phones->moveToSiblingPosition(0);   // become first sibling
$phones->moveToSiblingPosition(2);   // become third sibling
```

Position is 0-based, matching `moveTo()`, `TreeDiff`'s sibling positions, and the factory: position 0 is the first sibling, `count(siblings) - 1` is the last. Out-of-range positions throw `LogicException`. A root (no parent) throws `UnplacedNodeException` — roots have no sibling group to reorder within.

Equivalent to calling `$parent->reorderChildren(...)` with everyone else's order preserved.

## reorderChildrenBy — sort by column or closure

Sugar for "sort siblings alphabetically" and similar one-shots:

```php
$root->reorderChildrenBy('name');                          // by a column
$root->reorderChildrenBy(fn ($c) => strlen($c->name));     // by a closure
```

The closure receives the child model and returns any value Laravel's `Collection::sortBy()` accepts (scalar, array for multi-key sort, etc.). The resulting order feeds straight into `reorderChildren()`.

## reorderSiblings — static wrapper

Reads naturally at call sites that already have a parent variable in scope:

```php
Category::reorderSiblings($parent, [$c->id, $a->id, $b->id]);
```

Identical to `$parent->reorderChildren(...)`. Rejects a `$parent` of a different model class with `LogicException`.

## Things to know

### Reordering doesn't fire aggregate maintenance

Reordering doesn't change ancestry, so stored aggregate values on ancestors stay correct. The reorder path issues a raw UPDATE through the mutation builder, bypassing the `saving` / `saved` listener chain — the aggregate-maintenance hook never runs, and `assertAggregatesAreIntact()` continues to pass after a reorder.

### Concurrency: last write wins

Two clients reordering the same sibling group concurrently both compute their offsets from snapshots that may have been overwritten between their `SELECT` and their `UPDATE`. The result is always a **valid** order — one of the two requested — not a corrupt tree, but which one wins is non-deterministic. If you need a deterministic winner, wrap the call in `SELECT ... FOR UPDATE` on the parent row to serialise the two operations: the second client then computes its offsets from the first client's committed state, and the final order is the second client's request applied on top.

### The event surface

A non-identity reorder dispatches a single [`SiblingsReordered`](../reference/events.html) event with the parent, the post-reorder ID order, the number of rows the UPDATE touched, and the duration in milliseconds. Identity reorders (and empty-parent calls) emit no event because no UPDATE fires.

### One UPDATE, bounded by the subtree

The CASE-WHEN UPDATE has one branch per direct child whose `delta` is non-zero, and its `WHERE` is bounded by the parent's `lft` / `rgt` window. For a parent with `N` direct children and a total subtree size of `M`, the statement is one round-trip touching at most `M - 1` rows (the parent itself is excluded by the predicate).
