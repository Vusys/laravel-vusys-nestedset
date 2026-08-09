# Tree Repair

Production tables get corrupted — failed migrations, manual SQL surgery, bugs in old code. The repair toolkit lets you validate and rebuild:

```php
Category::isBroken();                       // bool
Category::countErrors();
// [
//   'invalid_bounds' => 0, 'duplicate_lft' => 2, 'duplicate_rgt' => 0,
//   'orphans' => 1, 'parent_bounds_mismatch' => 0, 'depth_mismatch' => 0,
//   'bounds_out_of_range' => 0, 'overlapping_bounds' => 0, 'even_bounds_width' => 0,
// ]

Category::fixTree();                        // rebuilds lft/rgt/depth from parent_id
// → TreeFixResult { nodesUpdated: 15, errors: [...counts after repair...] }
```

> [!NOTE]
> `nodesUpdated` is the **total size of the rebuilt set** (every row the repair walked in scope), not the number of rows that received a different `lft`/`rgt`/`depth`. Even on a clean tree it equals the row count. Don't use it as a "how much drift was there" signal — use the post-repair `errors` map, or compare `countErrors()` before and after. The field name predates the distinction and is preserved for backwards compatibility; expect a rename before 1.0.

On a scoped model, an anchor node is required so the repair stays inside one tree (this prevents accidental full-table walks on multi-million-row forests):

```php
MenuItem::isBroken();                       // ScopeViolationException — no anchor
MenuItem::isBroken($anyNodeFromThatMenu);   // OK — scoped to that menu

MenuItem::fixTree($anchor);                 // repair one menu's tree
```

**Anchored repair also works on unscoped models** — passing an anchor scopes the rebuild to that anchor's subtree rather than the whole table. Useful when you know one branch is broken and don't want to walk every root:

```php
Category::fixTree($electronics);   // rebuild Electronics + descendants only
Category::fixTree();               // rebuild every row in the table
```

Same `parent_id`-as-source-of-truth contract on both paths — the anchored rebuild reuses the same walk; it just starts from the anchor's children instead of from every root.

## What gets corrupted, what's auto-fixable, and how to avoid it

The package treats **`parent_id` as the source of truth**. `fixTree()` rebuilds `lft`/`rgt`/`depth` from a `parent_id` walk, so as long as `parent_id` describes the tree you actually want, every other column is recoverable.

### What the rebuild produces

After `fixTree()` returns, the tree is renumbered in canonical DFS pre-order — every `lft` strictly less than its `rgt`, every parent's `lft` / `rgt` strictly enclosing each child's, depth set to the path length from the root. Looking at the result, regardless of the corruption shape going in:

```ns-tree
Electronics
  Computers
    Laptops
    Desktops
  Phones
    iPhone
    Android
Books
  Fiction
  Non-fiction
```

The `lft` / `rgt` pill badges on each row show the dense slot ranges the rebuild assigns: Electronics (1..10), Computers (2..7), Books (11..16). Sibling order is determined by primary-key order under each parent — if you need a different sibling order, apply [`reorderChildren()`](../tree-operations/reordering.html) per parent after the rebuild.

| Corruption                                                                 | Detected by `countErrors()`?        | Repaired by `fixTree()`?               | Typical cause                                              |
| -------------------------------------------------------------------------- | ----------------------------------- | -------------------------------------- | ---------------------------------------------------------- |
| `invalid_bounds` (`lft >= rgt`)                                            | ✅                                   | ✅                                      | Raw `UPDATE` on `lft`/`rgt`; crashed transaction.          |
| `duplicate_lft` / `duplicate_rgt`                                          | ✅                                   | ✅                                      | Concurrent gap-shifts without locking; partial migration.  |
| `orphans` (`parent_id` → missing row)                                      | ✅                                   | ❌ — detected but not auto-repaired     | Hard `DELETE` of a parent without cascading.               |
| `parent_bounds_mismatch` (child bounds not inside parent's)                | ✅                                   | ✅                                      | Raw `UPDATE` on `parent_id` without rebuilding bounds.     |
| `depth_mismatch` (`depth` ≠ `parent.depth + 1`)                            | ✅                                   | ✅                                      | Raw edit that didn't also fix `depth`.                     |
| `bounds_out_of_range` (below-1 bound / cross-column collision)             | ✅                                   | ✅                                      | Raw `lft`/`rgt` literal outside the reserved range.        |
| `overlapping_bounds` (sibling intervals partially overlap without nesting) | ✅                                   | ✅                                      | Raw `lft`/`rgt` edit that left intervals intersecting.     |
| `even_bounds_width` (`rgt - lft` even — impossible in a valid set)         | ✅                                   | ✅                                      | Raw `lft`/`rgt` edit producing an even-width span.         |
| `parent_id` cycles                                                         | ❌ — not surfaced by `countErrors()` | ❌ — cycle members are silently skipped | Raw `UPDATE` on `parent_id` that bypassed Eloquent guards. |
| Aggregate drift (stored `articles_total` ≠ computed)                       | ✅ via `aggregateErrors()`           | ✅ via `fixAggregates()`                | Raw `UPDATE` on the source column.                         |

**Best practice in one rule:** mutate trees only through Eloquent on a `NodeTrait` model. Every `appendToNode`/`prependToNode`/`insertBeforeNode`/ `insertAfterNode`/`makeRoot`/`delete`/`forceDelete`/`restore` call is wrapped in a transaction and maintains every invariant. Most of the corruption categories above are reachable only by bypassing that surface.

See [Corruption Reference](corruption.html) for the full taxonomy with worked recovery recipes, diagnostic SQL for finding cycles, and `tests/Feature/Corruption/` for executable examples of every category.

## Limitations

### `fixAggregates()` assumes a structurally-sound tree

The fresh-aggregate read path relies on the nested-set invariant (`i.lft >= o.lft AND i.lft <= o.rgt` is equivalent to "i is a descendant of o" only when every row's `rgt` is consistent with its `lft`). Running `fixAggregates()` on a tree with `invalid_bounds` or `duplicate_lft`/`duplicate_rgt` errors can produce stored aggregates that disagree with what a healthy tree would compute.

The package's `fixTree()` runs `fixAggregates()` internally **after** structural repair, so the recommended recovery order is the one `fixTree()` enforces: structure first, aggregates second. Don't call `fixAggregates()` standalone on a tree you know is structurally broken.

### `fixTree($anchor)` rebuilds only the anchor's subtree

When you pass an anchor to `fixTree()`, the rebuild walks down from that anchor using `parent_id` and reassigns `lft`/`rgt`/`depth` for every reachable descendant. Rows **outside** the anchor's subtree are untouched.

If the anchor's subtree was corrupted in a way that changed its total size (e.g. orphans were force-deleted leaving phantom gaps, or descendants were added without `rgt`-shifting the ancestors), the rebuilt subtree may overlap surrounding rows in the same scope. In that case, fall back to the unanchored `fixTree()` which rebuilds every row in scope from `parent_id`.

The anchor itself must be **placed** — it needs a real `lft` to rebuild from. `fixTree($anchor)` on an unplaced node (`lft = 0`) throws `UnplacedNodeException` rather than write a subtree starting at `0` that collides with the real root; place the node first, or run the unanchored `fixTree()`.

`TreeFixResult::nodesUpdated` reports the number of rows the rebuild actually walked — the anchor's subtree for `fixTree($anchor)`, the whole scope for the unanchored form.
