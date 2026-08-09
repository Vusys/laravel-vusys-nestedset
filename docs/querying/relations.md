# Eloquent Relations

`NodeTrait` registers four relations on every node:

| Relation      | Type                    | Returns                                    |
| ------------- | ----------------------- | ------------------------------------------ |
| `parent`      | `BelongsTo`             | The immediate parent, or `null` for a root |
| `children`    | `HasMany`               | Immediate children (one level down)        |
| `ancestors`   | custom (eager-loadable) | Every node above this one                  |
| `descendants` | custom (eager-loadable) | Every node below this one                  |

```php
$laptops->parent->name;             // 'Computers'
$computers->children->pluck('name'); // ['Laptops', 'Desktops']

$laptops->ancestors->pluck('name');
// → ['Electronics', 'Computers']   — ordered root-to-parent

$electronics->descendants->pluck('name');
// → ['Computers', 'Laptops', 'Desktops', 'Phones', 'Android']
```

## Eager loading

The custom relations work with `with(...)` and load in two queries total — no N+1, regardless of how many rows you select:

```php
Category::with('ancestors')->get();      // breadcrumbs for every row, 2 queries
Category::with('descendants')->get();    // subtree for every row, 2 queries
```

`whereHas` and `withCount` work too:

```php
Category::whereHas('descendants', fn ($q) => $q->where('active', true))->get();
Category::withCount('descendants')->get();   // each row gets descendants_count
```

## Bounding the descendants relation

The descendants relation is **unbounded by default** — it pulls every descendant of every selected row. For trees with deep, wide subtrees this can be a lot more data than the UI needs. Bound the load to the first N levels by composing a `where` on the relation's `depth` column (which the trait already maintains):

```php
// Just children + grandchildren of $root (depth 1 + 2 relative to root)
$root->load([
    'descendants' => fn ($q) => $q->where('depth', '<=', $root->depth + 2),
]);

// Or on a top-level query — load every root with its first two levels
Category::with([
    'descendants' => fn ($q) => $q->where('depth', '<=', 2),
])->whereIsRoot()->get();
```

The bounded `WHERE` is served by the composite `(scope, lft, rgt, parent_id, …cover)` index on the `lft`/`rgt` range; the `depth` predicate is then a cheap filter over that already-narrow row set. (`depth` itself isn't in the index — add it to the migration's `cover` columns if you filter on depth heavily.)

Eager-loading `descendants` is also the recommended setup for [walking a subtree](walking.md) — the walker reads the loaded relation directly without issuing its own query.

## Combining with tree query scopes

Relations stack with the [tree-query scopes](queries.html) freely. This pattern is common for category-tree pages: load every root with its first two levels, ordered for display:

```php
$tree = Category::query()
    ->whereIsRoot()
    ->with(['descendants' => fn ($q) => $q->where('depth', '<=', 2)->defaultOrder()])
    ->defaultOrder()
    ->get();
```
