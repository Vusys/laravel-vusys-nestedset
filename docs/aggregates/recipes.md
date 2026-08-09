# Recipes

The aggregate primitives compose. A handful of common shapes:

## Status breakdown — one column per state

For a workflow column with a small enum of values, declare one filtered SUM and one filtered COUNT per state. Cheap delta path on every save; one extra `UPDATE` per state per mutation.

```php
#[NestedSetAggregate(column: 'open_tickets',   sum: 'tickets', filter: ['status' => 'open'])]
#[NestedSetAggregate(column: 'open_count',     count: true,    filter: ['status' => 'open'])]
#[NestedSetAggregate(column: 'closed_tickets', sum: 'tickets', filter: ['status' => 'closed'])]
#[NestedSetAggregate(column: 'closed_count',   count: true,    filter: ['status' => 'closed'])]
class Project extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

When a ticket flips from `open` to `closed`, the package fires a delta on `open_*` *and* on `closed_*` in the same `saved` event — `+/- ticket_value` on each pair, propagated to every ancestor in one `UPDATE`.

## Inclusive vs exclusive — totals including and below

Two declarations against the same source column gives you both "my subtree total" (inclusive) and "everything below me" (exclusive) without double-counting. UI screens use the exclusive value when they show *self* and *descendants total* side by side; the inclusive one when they show a single rollup.

```php
#[NestedSetAggregate(column: 'budget_inclusive', sum: 'budget')]
#[NestedSetAggregate(column: 'budget_below',     sum: 'budget', exclusive: true)]
class Department extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

`exclusive: true` excludes self from the rollup. A leaf reports `budget_below = 0`. A folder with three children each holding `budget = 100` reports `budget_inclusive = 300 + own_budget` and `budget_below = 300`.

Exclusive aggregates are maintained incrementally across every lifecycle hook (create, update, delete, restore, move). The chain-recompute path runs whenever a watched column dirties on save — cost shape is O(depth × subtree-size) per mutation, the same as the MIN/MAX extremum-lost branch. Mutations that don't touch a watched column skip the recompute entirely.

## Date-window roll-ups via raw filter

When the filter needs a SQL function or a comparison against something the equality / not-null forms can't express, drop down to `filterRaw`. Watched columns trigger an ancestor-chain recompute on save.

```php
#[NestedSetAggregate(
    column: 'recent_revenue',
    sum: 'revenue',
    filterRaw: 'closed_at >= CURRENT_DATE - INTERVAL 30 DAY',
    filterRawWatches: ['closed_at'],
)]
class Account extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

The watch on `closed_at` says "if this column changes on a save, the raw-filter column may need to be recomputed for the ancestor chain". A `name` change won't trigger the recompute; a `closed_at` change will.

Date-window filters have a second source of drift: the window slides every day. Schedule a periodic `fixAggregates()` (or `queueFixAggregates()`) to catch the rows that *would* re-enter or leave the window simply because of time passing — none of which fire a `saved` event.

## Weighted contributions via listener

When each row's contribution is a PHP expression — a product, ratio, lookup-driven value, anything that isn't a single column reference — declare a `TreeAggregateListener` and route through a listener aggregate. SUM is the common one but COUNT / MIN / MAX work too.

```php
final class RiskWeightedExposureListener implements TreeAggregateListener
{
    public function contribution(Model $node): int|float|null
    {
        return $node->exposure * ($node->risk_score / 100.0);
    }

    public function watchColumns(): array
    {
        return ['exposure', 'risk_score'];
    }
}

#[NestedSetAggregateListener(
    column: 'weighted_exposure',
    listener: RiskWeightedExposureListener::class,
    operation: AggregateFunction::Sum,
)]
class Position extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

The maintained column is a `decimal` (declare it via the migration's standard Blueprint helpers, not `nestedSetAggregate`, when you need a non-integer column type). Cast as `float` or `decimal:N` on the model.

## Conditional contribution via listener `null`

Listener `contribution()` can return `null` to exclude a row. This is the listener-side equivalent of a filter — useful when the inclusion test isn't expressible as a SQL predicate.

```php
final class ApprovedAmountListener implements TreeAggregateListener
{
    public function contribution(Model $node): ?int
    {
        // Only "approved" amounts roll up; everything else is excluded.
        return $node->status === 'approved' ? (int) $node->amount : null;
    }

    public function watchColumns(): array
    {
        return ['status', 'amount'];
    }
}
```

A `null` return excludes the row from the aggregate. For SUM the resulting value is the same as returning `0` (both add nothing); for COUNT / AVG / MIN / MAX the two differ — `null` skips the row entirely, while `0` still counts as a contributing row. Use `null` (not `0`) when the row genuinely doesn't qualify so a COUNT or AVG over the same listener gets the right denominator.

When the inclusion test is just a column equality or "this column is non-null", lift the predicate into a listener `filter:` / `filterNotNull:` parameter instead — the listener can stay generic and you can drive multiple per-condition aggregates from one listener class:

```php
#[NestedSetAggregateListener(
    column: 'approved_amount_sum',
    listener: AmountListener::class,
    operation: AggregateFunction::Sum,
    filter: ['status' => 'approved'],
)]
#[NestedSetAggregateListener(
    column: 'pending_amount_sum',
    listener: AmountListener::class,
    operation: AggregateFunction::Sum,
    filter: ['status' => 'pending'],
)]
class Invoice extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

See [Listeners → Filters](listeners.html#filters) for the full filter surface.

## Multiple Min/Max sliced by type

Filtered MIN/MAX gives you per-category extrema without a `GROUP BY` at read time. Useful for sidebar / dashboard widgets that show "lowest open priority", "highest urgent priority", etc.

```php
#[NestedSetAggregate(column: 'low_priority_min',  min: 'priority', filter: ['status' => 'open'])]
#[NestedSetAggregate(column: 'high_priority_max', max: 'priority', filter: ['status' => 'urgent'])]
class Issue extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

Each column gets its own cheap-delta / recompute behaviour independently — `low_priority_min` only triggers a recompute when the deleted/changed row's value matched the stored extremum AND the row's `status` was `open`.

## Ad-hoc fresh aggregates without declaration

`withFreshAggregates()` accepts inline `Aggregate` objects — no column needed on the model, no migration. Useful for one-off reports against an arbitrary predicate.

```php
$rows = Project::query()
    ->whereDescendantOf($rootBounds)
    ->withFreshAggregates([
        'p1_count'   => Aggregate::count()->filter(['priority' => 1]),
        'recent_sum' => Aggregate::sum('amount')
            ->filterRaw('created_at >= CURRENT_DATE - INTERVAL 30 DAY', []),
    ])
    ->get();
```

`filterRaw()` inlines its SQL verbatim — there is no parameter binding, so `?` placeholders are emitted literally. Pass only trusted constants written into the predicate; never user input.

The second argument to `filterRaw()` is the `$watches` list — required on the fluent factory (no default). For declared aggregates the registry's boot-time validation enforces that this list is populated or that you opt out via `filterRawNoColumnDependencies: true`; ad-hoc aggregates skip that validation since they have no boot pass, but the signature still requires the array. Pass `[]` for read-only / one-shot queries where maintenance doesn't apply.

The returned models have `p1_count` and `recent_sum` as computed attributes. Nothing's persisted; subsequent reads pay the correlated-subquery cost each time.

## Choosing the right form

| Need                                                   | Use                                                       |
| ------------------------------------------------------ | --------------------------------------------------------- |
| Sum/count/min/max/avg over a column, all rows          | unfiltered `#[NestedSetAggregate]`                        |
| Same, but only rows matching column = value            | `filter: ['col' => v]`                                    |
| Same, but only rows where a column is not null         | `filterNotNull: 'col'`                                    |
| Same, but predicate needs SQL functions or comparisons | `filterRaw: '...'` + `filterRawWatches: [...]`            |
| Contribution is a PHP expression                       | `TreeAggregateListener` + `#[NestedSetAggregateListener]` |
| Aggregate descendants only (not self)                  | `exclusive: true`                                         |
| One-off / ad-hoc / no column on the model              | `withFreshAggregates(['alias' => Aggregate::...])`        |
