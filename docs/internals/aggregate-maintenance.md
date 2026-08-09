# Aggregate Maintenance

The package can keep precalculated rollup columns — a SUM of every descendant's `tickets`, the cheapest `price` in a subtree, a distinct count, a weighted average — correct on every mutation, without recomputing the whole subtree each time. This is the most intricate subsystem in the package. This page walks how a declaration becomes stored columns and how those columns are maintained as the tree changes.

It is the implementation behind the user-facing [Aggregates](../aggregates/overview.html) section. The [Maintenance](../aggregates/maintenance.html) page already gives the per-mutation **cost model** (which family pays what); this page walks the **code and SQL** that produce those costs, so the two are complementary — cross-references point you to the cost tables rather than repeating them.

Two layers: `src/Concerns/HasNestedSetAggregates.php` (the lifecycle hooks, \~2900 lines) and `src/Aggregates/` (the strategies, function metadata, and declaration types).

## Three kinds of aggregate

- **SQL aggregates** — `#[NestedSetAggregate(column: 'tickets_total', sum: 'tickets')]`. The rollup is computed by SQL over a source column.
- **Listener aggregates** — `#[NestedSetAggregateListener(...)]` with a PHP `contribution(Model $node)` method per row.
- **Ad-hoc fresh aggregates** — `withFreshAggregates([...])`; no stored column, no maintenance, recomputed per read ([Query Engine](query-engine.html#fresh-aggregates-and-the-base-builder)).

This page is about the first two — the *maintained* kinds.

### Declaring an aggregate

The attribute form (`src/Attributes/NestedSetAggregate.php`) is preferred for static declarations and is repeatable:

```php
#[NestedSetAggregate(column: 'tickets_total', sum: 'tickets')]
#[NestedSetAggregate(column: 'tickets_count', count: true)]
#[NestedSetAggregate(column: 'tickets_avg',   avg: 'tickets')]
#[NestedSetAggregate(column: 'cheapest',      min: 'price')]
#[NestedSetAggregate(column: 'descendant_n',  count: true, exclusive: true)]
class Area extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

Each attribute must name **exactly one** function (`toDefinition()` throws otherwise) and resolves to an immutable `AggregateDefinition`. The fluent `Aggregate` factory (`src/Aggregates/Aggregate.php`) is the runtime/conditional escape hatch, used from a `nestedSetAggregates()` method override — it's also where the richer forms live (`filterRaw`, `weightedAvg`, `percentiles`, …):

```php
protected function nestedSetAggregates(): array
{
    return [
        Aggregate::sum('tickets')->into('tickets_total'),
        Aggregate::max('tickets')->into('tickets_max')->exclusive(),
        Aggregate::count()->filter(['active' => true])->into('active_count'),
    ];
}
```

## The four families

The maintenance strategy for a column is determined entirely by its function. `AggregateFunction` (`src/Aggregates/AggregateFunction.php`) declares which family each function belongs to via three predicates — `supportsDelta()`, `requiresChainRecompute()`, and `companionSet()`:

| Family                 | Functions                                                                                        | Strategy                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| **Delta-maintainable** | `Sum`, `Count`, `BitXor`                                                                         | `col = col ± Δ` per ancestor                                             |
| **Companion-derived**  | `Avg`, `Variance`, `Stddev`, `WeightedAvg`, `BoolOr`, `BoolAnd`, `GeometricMean`, `HarmonicMean` | delta the internal companions, rewrite the display column from a formula |
| **Recompute-only**     | `Min`, `Max`, `BitOr`, `BitAnd`, `DistinctCount`, `StringAgg`, `JsonAgg`, `JsonObjectAgg`        | `SELECT` then `UPDATE` the invalidated subset                            |
| **Fresh-read-only**    | `Median`, `Percentile`                                                                           | never stored                                                             |

`BitXor` is the standout — it is delta-maintainable on *both* insert and delete because XOR is self-inverse (`(x ^ a) ^ a = x`), so removing a contribution is the same operation as adding it. The function metadata records this directly:

```php
public function supportsDelta(): bool
{
    return match ($this) {
        self::Sum, self::Count, self::BitXor => true,
        // ... everything else => false
    };
}
```

The per-mutation cost of each family is tabulated in [Aggregates → Maintenance](../aggregates/maintenance.html#families).

## Companion columns

A companion-derived aggregate can't be maintained incrementally as a single stored value — but it *can* be if you store the right delta-maintainable pieces alongside it. `AVG` is the canonical case: store a `Sum` and a `Count` companion, and the displayed average is just `sum / count`, rewritten in the same statement that updates the companions. `companionSet()` declares the pieces each function needs:

```php
public function companionSet(): array
{
    return match ($this) {
        self::Avg => [
            new CompanionSpec('__sum', self::Sum),
            new CompanionSpec('__count', self::Count),
        ],
        self::Variance, self::Stddev => [
            new CompanionSpec('__sum', self::Sum),
            new CompanionSpec('__sum_sq', self::Sum, CompanionSourceTransform::Square),
            new CompanionSpec('__count', self::Count),
        ],
        self::WeightedAvg => [
            new CompanionSpec('__sum_wx', self::Sum, CompanionSourceTransform::TimesWeight),
            new CompanionSpec('__sum_w',  self::Sum, sourceOrigin: CompanionSourceOrigin::ParentWeight),
        ],
        // ... bool, geometric/harmonic mean ...
    };
}
```

The companion columns are allocated by the `nestedSetAggregate()` Blueprint macro ([Architecture → service provider](architecture.html#service-provider)) and resolved/auto-promoted by `AggregateRegistry`. Their names are the display column plus a suffix — `tickets_avg` gets `tickets_avg__sum` and `tickets_avg__count`; a variance column adds `__sum_sq`. Critically, the companion *storage type* is chosen to fit the maths: `__sum_sq` is a wide `DECIMAL(38,0)` (squares overflow bigint fast), the weighted-average companions are `DECIMAL` (they sum products of decimals), and the geometric/harmonic-mean companions are `DECIMAL(30,10)` because `LN(x)` and `1/x` are irrational. That storage-type decision lives in `NestedSetServiceProvider::companionAllocationsFor()`.

## Lifecycle hook ordering

Maintenance rides the Eloquent lifecycle events wired in `bootNodeTrait()` ([Architecture](architecture.html#lifecycle-wiring)). The hooks, all in `HasNestedSetAggregates`, fire in a fixed order relative to the structural SQL:

| Mutation             | Hooks (in order)                                                                          |
| -------------------- | ----------------------------------------------------------------------------------------- |
| Source-column update | `captureAggregateDeltas()` on `saving` → row UPDATE → `applyAggregateDeltas()` on `saved` |
| Insert (new node)    | row INSERT → `applyAggregateOnCreate()` on `created`                                      |
| Move (existing node) | `applyAggregateBeforeMove()` → structural SQL → `applyAggregateAfterMove()`               |
| Delete (hard/soft)   | cascade → `applyAggregateOnDelete()` → gap close                                          |
| Restore (soft)       | restore cascade → `applyAggregateOnRestore()`                                             |

The split between `saving` (capture) and `saved` (apply) exists because the delta has to be computed from the *old* and *new* source values while both are still available, but the `UPDATE` must run after the row itself is written. The capture step diffs the source column:

```php
$newSource = Numeric::asIntOrZero($this->getAttribute($source));
$oldSource = Numeric::asIntOrZero($this->getOriginal($source));
```

All of this is inside the one transaction the [mutation engine](mutation-engine.html) opened, so a failure anywhere rolls the rollups back with the structure.

## Delta maintenance — one `UPDATE` up the chain

`DeltaMaintenance::apply()` (`src/Aggregates/Strategy/DeltaMaintenance.php`) issues a single `UPDATE` constrained to the node's ancestor chain (and optionally self):

```php
$query = $connection->table($table)
    ->where($lftCol, '<=', $bounds->lft)
    ->where($rgtCol, '>=', $bounds->rgt);
```

That `lft <= node.lft AND rgt >= node.rgt` predicate is exactly `whereAncestorOrSelf` — the ancestors are the rows whose bounds contain this node. The `SET` clause adds the signed delta per column (`col ± Δ`), and for companion-derived columns it *also* rewrites the display column from the companions in the same statement. The clause ordering is load-bearing:

```php
$setExpressions = array_merge(
    self::buildAvgSetClauses($deltas, $avgs),          // derived display cols FIRST
    self::buildVarianceSetClauses($deltas, $variances),
    self::buildWeightedAvgSetClauses($deltas, $weightedAvgs),
    self::buildBoolSetClauses($deltas, $bools),
    self::buildMeanSetClauses($deltas, $means),
    self::buildDeltaSetClauses($deltas),               // raw companions AFTER
    self::buildExtremeSetClauses($extremes),
);
```

The derived display columns are emitted *before* the companion deltas because MySQL/MariaDB evaluate `SET` left-to-right with each assignment visible to later ones — so the display formula must read the companions' pre-update values and add the in-flight delta itself. The AVG clause shows the pattern:

```php
$clauses[$avgCol] = new TreeExpression(
    "(1.0 * ({$sumExpression})) / NULLIF(({$countExpression}), 0)",
);
```

where each `$sumExpression` is `sum_col + Δ`. The `1.0 *` forces decimal division (SQLite/PG truncate integer division otherwise), and `NULLIF(..., 0)` yields NULL on an empty subtree. PostgreSQL and SQLite evaluate all `SET` clauses against pre-update values regardless of order, so the same emission order is correct everywhere.

## Recompute maintenance — `SELECT` then `UPDATE` {#recompute-maintenance}

When no signed delta exists — MIN/MAX (a deleted extremum might be held by another row), raw filters, collection aggregates — the column routes through `RecomputeMaintenance::apply()` (`src/Aggregates/Strategy/RecomputeMaintenance.php`). It is two statements: a `SELECT` that recomputes each affected ancestor's value via an inner subtree subquery, then an `UPDATE` per touched ancestor.

```php
$selects[] = "(SELECT {$aggExpr} FROM {$table} AS inner_a "
    ."WHERE {$boundsClause}{$scopeJoin}{$exclusionClause}{$softInner}) AS {$alias}";
// ...
$where = "outer_a.{$lftCol} <= ? AND outer_a.{$rgtCol} >= ?";   // ancestor chain
```

It is *two* statements rather than one correlated-subquery `UPDATE` because MySQL forbids updating a table while selecting from it in a subquery; the SELECT-then- UPDATE shape works on all four backends. Three details make it efficient and correct:

### Cheap-skip

The caller passes the deleted/changed node's stored extremum (`$filterEquals`), ORed into the outer `WHERE`. Ancestors whose stored MIN/MAX *demonstrably didn't* match the changing value are never selected, so "delete a non-extreme row → recompute nothing".

### Inclusive vs exclusive bounds

The inner subquery uses `>= / <=` for inclusive aggregates and strict `> / <` for exclusive ones — `$boundsClause = $spec['inclusive'] ? "inner_a.{$lftCol} >= outer_a.{$lftCol} AND inner_a.{$rgtCol} <= outer_a.{$rgtCol}" : "inner_a.{$lftCol} > outer_a.{$lftCol} AND inner_a.{$rgtCol} < outer_a.{$rgtCol}";`

### `excludeBounds`

On the before-move hook the moving subtree is still physically present, so its rows are excluded from the scan (`AND NOT (inner_a.lft >= … AND inner_a.rgt <= …)`) to reflect the post-move-but-pre-SQL state.

Whether the SELECT takes a `FOR UPDATE` lock is governed by `aggregate_locking` — see [Concurrency & Transactions](concurrency.html#aggregate-locking).

## Numeric type preservation — the footgun {#type-preservation}

The single subtlest bug surface in the subsystem is how stored aggregate columns are read back when their contribution is removed (delete/move/restore). The helpers in `src/Aggregates/Numeric.php` encode three different contracts — `asIntOrZero()`, `asNumericOrNull()`, and `asNumericOrZero()` — and **picking the wrong one silently corrupts the rollup**.

The rule the code follows: a **source column** that is structurally an integer is read with `asIntOrZero()`, but a **stored aggregate column** is read with `asNumericOrZero()`, because companion sums can be fractional. You can see both in `applyAggregateOnDelete()`:

```php
if ($definition->function === AggregateFunction::Sum
    || $definition->function === AggregateFunction::Count) {
    // Preserve numeric type — Sum companions of WeightedAvg /
    // GeometricMean / HarmonicMean hold decimal sums (sum_wx,
    // sum_log, sum_recip) that numeric() would truncate to 0
    // or int-cast away the fractional part.
    $value = Numeric::asNumericOrZero($this->getAttribute($definition->column));
    if ($value != 0) {
        $deltas[$definition->column] = -$value;
    }
    continue;
}
```

`asNumericOrZero()` keeps `"10"` an int but `"10.5"` a float:

```php
return str_contains($string, '.') || str_contains($string, 'e') || str_contains($string, 'E')
    ? (float) $value
    : (int) $value;
```

If a `__sum_recip` companion holding `4.2` were read with `asIntOrZero()`, the ancestor would be decremented by `4` instead of `4.2` and its harmonic-mean display column would drift. (A related trap: a NULL stored MIN/MAX read with `asNumericOrZero()` collapses to `0` and propagates as a fake candidate extreme — the regression test in `tests/Feature/Aggregates/FilteredDeltaMaintenanceTest.php` pins this.) The `Numeric` docblock calls this out explicitly as a known footgun.

## Moves — subtracting from old, adding to new {#moves}

A move re-parents a subtree, so its contribution must leave the old ancestor chain and join the new one. The [mutation engine's seams](mutation-engine.html#the-aggregate-seams) call `applyAggregateBeforeMove()` (pre-SQL, old chain) and `applyAggregateAfterMove()` (post-SQL, new chain). The before hook reads the moving subtree's stored contribution and applies a *negative* delta to the old ancestors (using `asNumericOrZero()` again for the same type-preservation reason); the after hook applies the positive delta to the new ancestors. For recompute-only columns the before hook uses `excludeBounds` so the old chain recomputes as if the subtree were already gone.

## Soft deletes & restore {#soft-deletes}

Soft-deleted trees use **snapshot semantics**. When a subtree is soft-deleted, the delta `UPDATE` decrements its live ancestors once, and from then on every maintenance `UPDATE` carries `WHERE deleted_at IS NULL` — so trashed ancestors are frozen at their trash-time values:

```php
if ($softDeletedColumn !== null) {
    $query->whereNull($softDeletedColumn);
}
```

The recompute path mirrors this on both sides of the join — trashed *ancestors* stay frozen (`outer_a.deleted_at IS NULL`) and trashed *descendants* don't contribute (`inner_a.deleted_at IS NULL`).

Restore is *not* a simple re-add, because descendants may have been restored independently (a partial restore). So `applyAggregateOnRestore()` recomputes the restored subtree from its now-live descendants and then chain-recomputes the ancestors, rather than blindly adding back a stored total — the safe choice that handles partial restores correctly. The cascade ordering (descendants un-trashed *before* the aggregate hook runs) is enforced in `bootNodeTrait()`'s `restored` listener so chain recomputes see the final live state.

## Repair — `fixAggregates()` {#repair-fixaggregates}

Stored rollups can drift (a mass-assignment, a raw `UPDATE`, a bug). `fixAggregates()` recomputes every stored column from source and writes back the differences — the aggregate analogue of [`fixTree()`](repair.html). It compares stored vs freshly computed values (via `AggregateDiffer` / `AggregateValueComparator` in `src/Query/Aggregates/Maintenance/`) and bulk-writes corrections. `fixTree()` calls it as a final step so one repair call fixes structure *and* rollups, against the post-repair tree. There is also a chunked + queued form (`queueFixAggregates()`) for large tables. See [Repairing Aggregates](../maintenance/fix-aggregates.html).

## Deferred maintenance

For batch mutations, per-save maintenance is wasteful — every save pays the ancestor-chain `UPDATE`. `withDeferredAggregateMaintenance(Closure $work)` sets a depth guard (`self::$deferredDepth`) that every hook checks first:

```php
public function applyAggregateOnDelete(): void
{
    if (self::$deferredDepth > 0) {
        return;
    }
    // ...
}
```

Inside the closure the structural mutations still run (lft/rgt stay correct) but aggregate hooks no-op; a single `fixAggregates()` runs once the closure exits. This is exactly what `bulkInsertTree()` uses internally ([Bulk Insertion](../tree-operations/bulk-insertion.html)).

## Where to go next

[Integrity & Repair](repair.html) covers the structural side of the same repair machinery, and [Concurrency & Transactions](concurrency.html) explains the locking that keeps the recompute path correct under concurrent writers.
