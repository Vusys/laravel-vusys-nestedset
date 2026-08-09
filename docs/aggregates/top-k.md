# Top-K aggregate

`topK` stores the *K rows with the largest ranking value* anywhere in a node's subtree, as a JSON array of `[source_value, by_value]` pairs. It's the "best sellers per category" / "highest-revenue subordinates per manager" shape — a query that's awkward and expensive to compute on read, but cheap to store as a maintained column.

| Kind   | Stored column type | Empty subtree | Maintenance            |
| ------ | ------------------ | ------------- | ---------------------- |
| `topK` | `json` (nullable)  | `NULL`        | Full subtree recompute |

`topK` is **recompute-only**: a single deletion or move can promote a row that the stored list never tracked, so no signed delta exists. Every contributing mutation triggers a full subtree recompute over the ancestor chain (same path as the [collection aggregates](text-and-json.html)).

## Quick example

```php
use Vusys\NestedSet\Attributes\NestedSetAggregate;

#[NestedSetAggregate(column: 'top_revenue_products', topK: 'product_id', k: 5, by: 'revenue')]
class Department extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

Storage shape on each department row:

```json
[
  [481, 19200],
  [266, 17850],
  [319, 12340],
  [104, 9870],
  [820, 8430]
]
```

— the top 5 descendant products by revenue, in descending order.

## Parameters

- `topK` (attribute) / first argument to `Aggregate::topK(...)` — the column whose **value** goes in position `0` of each stored pair. Typically the row's `id` or a foreign-key column you want to identify by.
- `k` — how many entries to keep. Must be ≥ 1.
- `by` — the column used to **rank** the rows. Defaults to the source column when omitted (`topK: 'price', k: 5` ⇒ "top 5 prices" — the source ranks itself).

## Method-override form

```php
use Vusys\NestedSet\Aggregates\Aggregate;

protected function nestedSetAggregates(): array
{
    return [
        Aggregate::topK('product_id', 5, by: 'revenue')->into('top_revenue_products'),
    ];
}
```

## Migration

```php
$table->nestedSetAggregate('top_revenue_products', type: 'top_k');
```

The `top_k` type emits a nullable JSON column — `jsonb` on PostgreSQL, `JSON` on MySQL/MariaDB, TEXT on SQLite — same shape and per-backend dispatch as `jsonAgg`.

Add a `'top_revenue_products' => 'array'` entry to the model's `$casts` and Eloquent will decode the JSON to a PHP array on read.

## Behaviour

- **Ordering**: by `by` column DESC, with the source column DESC as a tiebreaker (so ties resolve deterministically across all four supported backends).
- **NULL `by` values**: rows with `NULL` in the `by` column are excluded — every backend orders NULL unpredictably and silently letting them in would make the stored result non-deterministic.
- **Empty subtree**: NULL (the column is nullable).
- **Filters**: the standard `filter`, `filterNotNull`, and `filterRaw` modifiers all apply — the ranking pool is restricted to matching rows.

```php
#[NestedSetAggregate(
    column: 'top_active_products',
    topK: 'product_id',
    k: 5,
    by: 'revenue',
    filter: ['status' => 'active'],
)]
```

## Fresh-read use

`topK` also works inside `withFreshAggregates()` for ad-hoc reads:

```php
Category::query()
    ->withFreshAggregates(['top3' => Aggregate::topK('id', 3, 'revenue')])
    ->get();
```

## Limitations

- **No delta path.** Insert / update / delete / move all route through full subtree recompute. For shallow trees this is cheap; for deep trees, factor in the cost when planning aggregate budgets.
- **K must be a compile-time constant per declaration.** Dynamic K across rows isn't expressible.
- **No runner-up tracking.** The maintained column holds exactly K entries, not K+1. (The optimisation outlined in `FUTURE_AGGREGATES.md` for delta-maintained Top-K with a runner-up safeguard is a future direction; the recompute-only shape implemented here doesn't need it.)
