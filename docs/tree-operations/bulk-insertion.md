# Bulk Tree Insertion

Seeding a large tree by calling `appendToNode()->save()` per node is O(N²) — each save shifts every post-insertion-point row through a CASE-WHEN UPDATE, so the gap-shift cost piles up over the whole insert. `bulkInsertTree()` collapses that to one `makeGap` plus one `fixAggregates`, while keeping every Eloquent guarantee (events, mutators, casts, mass-assignment) and returning fully hydrated models:

```php
$root = new Category(['name' => 'All categories']);
$root->saveAsRoot();
$root = $root->refresh();

[$electronics, $computers, $laptops, $desktops, $phones, $books] = Category::bulkInsertTree([
    ['name' => 'Electronics', 'children' => [
        ['name' => 'Computers', 'children' => [
            ['name' => 'Laptops'],
            ['name' => 'Desktops'],
        ]],
        ['name' => 'Phones'],
    ]],
    ['name' => 'Books'],
], appendTo: $root);

$electronics->name;                            // 'Electronics' — mass-assigned via $fillable
$electronics->wasRecentlyCreated;              // true
$laptops->parent_id === $computers->getKey();  // true
```

The nested-array payload above translates directly to this tree under `$root` after the call. The `lft` / `rgt` slots are assigned in one pass — no incremental gap-shifting, no N round-trips:

```ns-tree
All categories
  Electronics
    Computers
      Laptops
      Desktops
    Phones
  Books
```

The returned array is in depth-first pre-order — the same walk order as `defaultOrder()` once the rows are persisted. Top-level entries come first; each entry is immediately followed by its descendants. The `lft` / `rgt` badges on each row above show the slot range each subtree occupies; reads the package issues against this tree (`whereDescendantOf`, `ancestors()`, etc.) are all `BETWEEN` queries over those ranges.

## Seeding new roots (unscoped models)

Passing `appendTo: null` on an **unscoped** model seeds the input as new root(s) — the package starts one past the current `MAX(rgt)` so the new trees never overlap existing ones. Useful for first-run fixtures or admin-imported batches that should each become their own top-level tree:

```php
[$site1, $site2] = Category::bulkInsertTree([
    ['name' => 'Site 1', 'children' => [['name' => 'Home']]],
    ['name' => 'Site 2'],
]);

$site1->isRoot();   // true
$site2->isRoot();   // true
```

To seed new roots on a **scoped** model, pass the target scope explicitly via the `$scope` argument — `Model::bulkInsertTree($tree, null, ['tenant_id' => 7])` — and the package scopes the root placement to that tenant. Passing neither an anchor nor `$scope` on a scoped model still throws `ScopeViolationException` (the scope-column values have to come from somewhere).

Each row goes through a normal `save()`, so per-row `creating` / `saving` / `created` / `saved` events still fire, every cast applies, observers run, mass-assignment guards are respected. The only operations the package does on top of `save()` are the one-shot `makeGap` (replaces N gap-shifts) and the deferred `fixAggregates` at the end of the call (replaces N aggregate-ancestor UPDATEs).

## Performance

| Backend | N=100 (vs naive)   | N=1000 (vs naive)    |
| ------- | ------------------ | -------------------- |
| SQLite  | 18ms (3.2× faster) | 224ms (3.2× faster)  |
| MySQL 8 | 44ms (3.0× faster) | 302ms (6.0× faster)  |
| MariaDB | 48ms (6.6× faster) | 451ms (11.9× faster) |

The win widens as N grows because the naive path's gap-shift cost is O(N²). At N=10,000 on MariaDB the naive loop runs for many minutes; `bulkInsertTree` scales roughly linearly.

## Event-free seeding

If you specifically need event-free seeding (e.g. backfilling 100K rows from a CSV with no observer side effects), the standard Laravel escape hatch composes:

```php
Model::withoutEvents(static fn () => Category::bulkInsertTree($rows, appendTo: $root));
```

## Constraints

### Structural columns are reserved

Rows must not contain `lft`, `rgt`, `depth`, `parent_id`, or the primary key — those are computed by the package.

### Scope columns are silently overwritten with the anchor's values

On scoped models, every inserted row's scope-column attributes are replaced with the values read off `$appendTo` regardless of what the input row contains. This is by design (a bulk insert into one anchor's subtree always belongs to that anchor's scope) but worth knowing — passing `['tenant_id' => 99]` in a row when `$appendTo->tenant_id === 7` produces a row with `tenant_id = 7`, not an error.

### Scoped models require scope context

Models declared with `#[NestedSetScope]` or `getScopeAttributes()` need scope-column values from somewhere — either an `$appendTo` anchor (its scope is copied onto every inserted row) **or**, when seeding new roots with no anchor, an explicit `$scope` argument (`bulkInsertTree($tree, null, ['tenant_id' => 7])`). Passing neither throws `ScopeViolationException`.

### Transactional

Wrapped in a transaction; if any per-row `save()` throws, the gap-open and any prior inserts roll back together.

## Test fixtures and seeders

For tests and seeders, prefer the [Factory Tree Builder](../reference/factories.md) — a factory mixin (`BuildsNestedSetTrees`) that hands `bulkInsertTree` the same payload you'd assemble manually, plus factory state, per-row customisation, and `afterCreating` integration. `Category::factory()->tree(depth: 3, branching: 2)->create()` builds a 15-node tree in one call with the same three-statement budget.
