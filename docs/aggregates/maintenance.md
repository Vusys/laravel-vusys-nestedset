# Maintenance

Aggregates ride the package's existing lifecycle events. The cost of any single mutation depends on which family the column belongs to.

## One mutation, traced through the tree

Before the family taxonomy, here's the shape every delta-maintained aggregate follows. Take a budget tree with a maintained `cost_total = SUM(cost)`:

```ns-tree
Engineering
  People
    Salaries {cost=26000}
    Bonuses {cost=2000}
  Tools
    SaaS {cost=2500}
    Hardware {cost=1500}
```

Each node's `Σ cost` is the stored `cost_total` rolled up over its subtree. A single source-column update doubles Bonuses' cost — done through a **model instance**, so the `saving`/`saved` hooks fire and maintenance runs:

```php
$bonuses->cost = 4000;
$bonuses->save();
```

> [!WARNING]
> Maintenance is driven by Eloquent model events. A bulk query-builder write — `Account::query()->where(...)->update(['cost' => 4000])` — fires **no** model events, so the aggregate columns are **not** updated and drift silently. After any bulk update of a source column, run [`fixAggregates()`](drift.html) (anchored, on scoped models) to resync.

After the write, every **ancestor** of Bonuses has `cost_total += 2000`. Sibling rows (`Salaries`, `Tools`, `SaaS`, `Hardware`) and the unrelated `Engineering` subtree are untouched — the delta only flows up the ancestor chain:

```ns-tree
Engineering
  People
    Salaries {cost=26000}
    Bonuses {cost=4000}
  Tools
    SaaS {cost=2500}
    Hardware {cost=1500}
```

That's the **delta-maintainable** path in code form: one `UPDATE cost_total = cost_total + 2000 WHERE lft <= Bonuses.lft AND rgt >= Bonuses.rgt`, one round-trip, work proportional to the ancestor depth (2 rows: `People`, `Engineering`). The same shape applies to inserts (Δ = `+new.cost`), deletes (Δ = `-old.cost`), moves (Δ on old chain, +Δ on new chain), and soft-delete cascades. The families below describe what changes when the column *isn't* directly delta-maintainable — but the ancestor-chain pattern is the constant across all of them.

## Families

| Family                 | Members                                                                                          | Per-mutation cost                                                                                                                                                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Delta-maintainable** | `Sum`, `Count`                                                                                   | One `UPDATE col = col ± Δ` per ancestor chain. Constant work per row.                                                                                                                                                                                  |
| **Companion-derived**  | `Avg`, `Variance`, `Stddev`, `WeightedAvg`, `BoolOr`, `BoolAnd`, `GeometricMean`, `HarmonicMean` | Delta on the internal companions; the user-facing display column is rewritten in the same `UPDATE` from a formula over those companions. Single statement per ancestor chain.                                                                          |
| **Extremum-recompute** | `Min`, `Max`                                                                                     | Cheap delta when the change can't possibly invalidate the stored extremum (ascending update on `Max`, descending update on `Min`, insert of a non-extreme row). Otherwise SELECT-then-UPDATE recompute of the invalidated subset.                      |
| **Chain-recompute**    | `BitOr`, `BitAnd`, `BitXor`, `DistinctCount`, `StringAgg`, `JsonAgg`, `JsonObjectAgg`            | Full subtree scan up the ancestor chain on every mutation that touches the source column. No signed delta is used (for `BitOr`/`BitAnd` none exists — a removed bit might still be held by another row; `BitXor` recomputes too for one uniform path). |
| **Fresh-read-only**    | `Median`, `Percentile`                                                                           | No maintenance — no stored column. Each read is one correlated subquery per outer row per percentile. See [Quantiles](quantiles.html).                                                                                                                 |

## Per-mutation table

The most common shapes:

| Mutation                   | Delta-maintainable / companion-derived                | Min / Max                                                                                            | Bitwise (`BitOr` / `BitAnd` / `BitXor`) / collection |
| -------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Insert leaf                | 1 delta UPDATE on ancestor chain                      | 1 cheap-update; recompute only when the new leaf could become the extremum                           | Chain recompute on ancestor chain                    |
| Source-column update       | 1 delta UPDATE                                        | 1 delta when the change can't invalidate the stored extremum; SELECT-then-UPDATE recompute otherwise | Chain recompute                                      |
| Delete                     | 1 delta UPDATE (negated Δ)                            | 1 delta when the deleted row wasn't the extremum; recompute otherwise                                | Chain recompute                                      |
| Move (`appendToNode` etc.) | 1 delta on old chain + 1 on new chain                 | Old-chain recompute when the moved subtree contained the extremum; cheap delta on the new chain      | Recompute on both chains                             |
| Soft-delete + restore      | Delta subtract on cascade, delta re-add on `restored` | Same as delete/insert                                                                                | Recompute on both                                    |

`MIN`/`MAX` use a SELECT-then-UPDATE recompute path when the change may have invalidated the stored extremum — the same path applies to raw-filter columns whose `filterRawWatches` columns dirty on save. Concurrency between the SELECT and UPDATE is governed by the `nestedset.aggregate_locking` config flag (`'auto'` / `'always'` / `'never'`; see [Configuration → aggregate\_locking](../reference/config.html#aggregate_locking)).

The chain-recompute families pay O(ancestor-depth × subtree-size-at-each-ancestor) per mutation. If your write path is hot and you have one of these declared on a deep tree, consider [deferred maintenance](../maintenance/fix-aggregates.html#deferred-maintenance-for-batch-mutations) — every save inside the closure skips the per-row aggregate update; one `fixAggregates()` runs at the end.

## Integrity tooling

Mirrors the tree-repair API:

```php
Category::aggregateErrors();
// ['articles_total' => 0, 'articles_count_all' => 0, 'articles_avg' => 0, ...]

Category::aggregatesAreBroken();    // bool

Category::fixAggregates();
// → AggregateFixResult { totalRowsUpdated: 0, perColumn: [...] }
```

`fixTree()` runs `fixAggregates()` as a final step — corrupted lft/rgt plus drifted aggregates are repairable in one call. The result carries the aggregate stats alongside the tree stats:

```php
$result = Category::fixTree();
$result->nodesUpdated;       // tree side — total size of the rebuilt set
$result->errors;             // post-repair tree errors
$result->aggregatesFixed;    // AggregateFixResult — null on no-aggregate models
```

Scoped models require an anchor on `aggregateErrors`, `aggregatesAreBroken`, and `fixAggregates` (same as `fixTree`).

## Observability

Aggregate maintenance fires typed events on Laravel's event bus — `FixAggregatesCompleted`, `FixAggregatesChunkCompleted`, `FixAggregatesJobDispatched`, `DeferredMaintenanceStarting`, `DeferredAggregateMaintenanceCompleted`, `NodeAggregatesRecomputed`, `NestedSetAggregateChanged`, `AggregateDriftDetected`, `AggregateMaintenanceFailed`.

`NestedSetAggregateChanged` is the per-(row, column) CDC change feed for maintained aggregates — useful for mirroring to Redis / Kafka / search indexes without polling. It is **opt-in by listener presence**: the firing site short-circuits when nobody subscribes, so the package pays no extra read-side cost when the feed is unused.

See the [Events](../reference/events.html#aggregate-maintenance) page for the full payload reference, recipes, and queue-safety notes.

## Adding aggregates to an existing model

1. Add `#[NestedSetAggregate(...)]` declarations to the model class.
2. Add `$table->nestedSetAggregate('col_name', type: ...)` to a new migration; run it. The companion columns appear automatically — see [Setup → type: reference](setup.html#type-reference) for the complete shape table.
3. Add the matching cast to `$casts`.
4. Run `YourModel::fixAggregates()` once to backfill stored values from the source data. On scoped models, run per anchor.
5. Deploy.

After the backfill, every subsequent mutation through Eloquent keeps the stored values current.
