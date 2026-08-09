# Median, Percentile, and Quantiles

Quantile aggregates are **read-only** — they're available exclusively through `withFreshAggregates()` and cannot be declared as stored columns. The package supports `median()`, arbitrary `percentile($p)`, plus the helper bundlers `percentiles([...])` and `quartiles()`.

```php
use Vusys\NestedSet\Aggregates\Aggregate;

$rows = Category::query()
    ->withFreshAggregates([
        'price_median' => Aggregate::median('price'),
        'price_p95'    => Aggregate::percentile('price', 0.95),
    ])
    ->get();

$rows->first()->price_median;   // float (linear-interpolated)
$rows->first()->price_p95;
```

The `percentiles()` helper produces a map of `alias => Aggregate` that spreads straight into `withFreshAggregates()`, useful when you want several points in one read:

```php
$rows = Category::query()
    ->withFreshAggregates([
        ...Aggregate::percentiles('price', ['p25' => 0.25, 'p50' => 0.5, 'p95' => 0.95]),
    ])
    ->get();

$rows->first()->p25;
$rows->first()->p50;
$rows->first()->p95;
```

`quartiles()` is sugar for `percentiles($source, ['q1' => 0.25, 'median' => 0.5, 'q3' => 0.75])`:

```php
Category::query()
    ->withFreshAggregates([...Aggregate::quartiles('price')])
    ->get();
```

## Why these are fresh-read-only

Quantiles can't be maintained delta-wise. Inserting or deleting a single row can shift the median to anywhere in the subtree's value range, so there's no signed contribution an ancestor can apply incrementally — every read needs to scan the subtree's source values, sort them, and interpolate.

Declaring `median` or `percentile` as a stored column fails at the attribute boundary so the mistake surfaces at boot, not at write time:

```php
#[NestedSetAggregate(column: 'price_median', median: 'price')]  // throws:
class Category extends Model implements MaintainsTreeAggregates { use NodeTrait; }
// AggregateConfigurationException — median() and percentile() are
// recompute-only and cannot be stored as precalculated aggregate
// columns. Use withFreshAggregates() for on-demand quantile reads.
```

There's no `nestedSetAggregate('col', type: 'median')` migration shape either — the type doesn't exist.

## Linear interpolation semantics

Both `median()` and `percentile($p)` use **linear interpolation** — the same definition PostgreSQL's `PERCENTILE_CONT` implements. NULL source values are excluded from the ordered set (matching PG's ordered-set aggregate behaviour); the package's other backends enforce this with an explicit `IS NOT NULL` predicate so all four behave identically.

For a sorted set of `n` non-null values and a percentile point `p ∈ [0, 1]`:

```text
pos      = p · (n − 1)
result   = (1 − frac) · values[floor(pos)] + frac · values[ceil(pos)]
where frac = pos − floor(pos)
```

A subtree with a single value returns that value. An empty subtree (no non-null source rows under the node) returns NULL.

## Backend dispatch

The package emits the right SQL per driver — same semantics across all four:

| Backend       | Shape                                              | Notes                                                                                                              |
| ------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| PostgreSQL    | `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY src)`   | Native ordered-set aggregate.                                                                                      |
| MySQL, SQLite | `ROW_NUMBER()` window subquery + interpolation     | Picks the two flanking values and interpolates inline.                                                             |
| MariaDB       | `JSON_ARRAYAGG` + `JSON_VALUE` correlated subquery | MariaDB rejects derived tables that reference the outer query's columns; the JSON form is the portable workaround. |

Each quantile reads as one correlated subquery per outer row — so requesting `n` percentiles over `m` outer rows costs `n × m` subqueries. A leaf row's quantile short-circuits to the source value itself (no subquery), so a tree where you only care about non-leaf nodes pays predictably less than a flat table of leaves.

## Filtering

Quantile aggregates accept the same filter modifiers as the other fresh reads — `filter`, `filterNotNull`, `filterRaw`:

```php
Category::query()
    ->withFreshAggregates([
        'in_stock_median' => Aggregate::median('price')->filter(['in_stock' => true]),
        'recent_p95'      => Aggregate::percentile('viewed_at', 0.95)
            ->filterRaw("viewed_at > '2020-01-01'", ['viewed_at']),
    ])
    ->get();
```

`filterRaw()` inlines its SQL verbatim — write the predicate with literal values, not `?` bound parameters, since the package does not parameter-bind filter SQL (see [Filtered Aggregates → security note](filtered.html#filter-forms)). The `$watches` list is required by the signature but doesn't drive maintenance here — there's nothing to maintain — pass `[]` if the predicate references no columns, or list every referenced column otherwise.

## When to reach for quantiles vs. stored aggregates

| Question                               | Use                                                      |
| -------------------------------------- | -------------------------------------------------------- |
| Average / total / count                | Stored aggregate column (delta-maintained).              |
| Min / max / variance / stddev          | Stored aggregate column.                                 |
| Median, p95, IQR, arbitrary percentile | `withFreshAggregates([... median / percentile ...])`.    |
| Quartile breakdown for a dashboard     | `withFreshAggregates([...Aggregate::quartiles('col')])`. |

Stored aggregates are effectively free per read; quantiles cost one correlated subquery per outer row per requested percentile. For dashboards that surface percentiles next to means, ship both in a single `withFreshAggregates()` call — the percentile cost dominates either way, so add the stored-column reads to the same query rather than paying two round-trips.

## Validation

- `percentile($source, $p)` requires `$p` in `[0.0, 1.0]`; outside that range throws `AggregateConfigurationException` at the fluent call.
- `percentiles($source, $points)` requires a non-empty `$points` map; every value must be in `[0.0, 1.0]` and every key must be a non-empty string alias.
- Source column must be a non-empty string for all four factories.

All validation fires at declaration time, not at query execution, so configuration errors surface synchronously when you build the call rather than when the page renders.
