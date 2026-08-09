# Geometric and Harmonic Mean

Two delta-maintainable mean aggregates for source columns where the arithmetic mean is the wrong question. Both are companion-derived (same machinery as `Avg`, `Variance`, `Stddev`, `WeightedAvg`) so a read is a plain column read and a write is one delta UPDATE per ancestor chain.

- **Geometric mean** — `EXP(Σ LN(source) / n)`. Reach for it when source values multiply rather than add (compound growth rates, ratios, scale factors). The geometric mean of two numbers `a` and `b` is `sqrt(a · b)` — closer to the smaller of the two than the arithmetic mean.
- **Harmonic mean** — `n / Σ(1/source)`. Reach for it when source values are rates over a common unit and you want the average rate (parallel resistance, average speed across equal distances). The harmonic mean weighs small values more heavily than the arithmetic mean.

## When the arithmetic mean is wrong

A portfolio of investments, each row carrying its yearly growth rate as a multiplier (`1.10` = +10%, `0.80` = −20%):

```ns-tree
Portfolio
  Tech ETF {growth_rate=1.30}
  Bonds {growth_rate=0.90}
  Real Estate {growth_rate=1.20}
  Commodities {growth_rate=0.80}
```

The arithmetic mean of those rates is `1.05` — but that's the wrong answer for "average compound growth across these positions." Compounding is multiplicative, so the right mean is the **geometric** one: `(1.30 · 0.90 · 1.20 · 0.80)^(1/4) ≈ 1.030`. The portfolio's *actual* compound rate per position is 3.0%, not 5%. That gap widens as variance grows — the arithmetic mean over-reports compound growth whenever the underlying values are dispersed.

The maintained `growth_geomean` column on `Portfolio` holds 1.030 directly; the package writes that value from the `__sum_log` and `__count` companions on every relevant mutation. One delta UPDATE per ancestor chain; no PHP-side arithmetic.

A harmonic example: average speed across road segments of equal length (a key fact — equal distances, varying speeds). A driver covers each segment at a different speed:

```ns-tree
Commute
  Highway {speed_kph=120}
  City {speed_kph=40}
  Suburban {speed_kph=60}
```

Arithmetic mean: `73.3 kph`. But that overstates the actual average speed across the commute — the slow segment dominates the trip time. The harmonic mean handles the rate-aggregation correctly: `3 / (1/120 + 1/40 + 1/60) = 60 kph`. That's the speed you'd see if you divided total distance by total time. The maintained `speed_harmean` column on `Commute` is what the package stores.

```php
use Vusys\NestedSet\Attributes\NestedSetAggregate;

#[NestedSetAggregate(column: 'growth_geomean', geometricMean: 'growth_rate')]
#[NestedSetAggregate(column: 'speed_harmean',  harmonicMean:  'speed_kph')]
class Segment extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;
}
```

Or via the method-override form:

```php
protected function nestedSetAggregates(): array
{
    return [
        Aggregate::geometricMean('growth_rate')->into('growth_geomean'),
        Aggregate::harmonicMean('speed_kph')->into('speed_harmean'),
    ];
}
```

## Migration

```php
$table->decimal('growth_rate', 10, 4)->nullable();
$table->decimal('speed_kph',   10, 4)->nullable();

$table->nestedSet();
$table->nestedSetAggregate('growth_geomean', type: 'geometric_mean');
$table->nestedSetAggregate('speed_harmean',  type: 'harmonic_mean');
```

Each declaration allocates the user-facing display column **plus** two delta-maintainable companions:

| Display column       | Companions allocated                     |
| -------------------- | ---------------------------------------- |
| `geometricMean(src)` | `*__sum_log` (= `Σ LN(src)`), `*__count` |
| `harmonicMean(src)`  | `*__sum_recip` (= `Σ 1/src`), `*__count` |

The display column is a nullable `decimal(12, 4)`. The `__sum_log` / `__sum_recip` companions are `decimal(30, 10)` — wider fractional precision than the standard `decimal_sum` companion (4 digits) because `LN(x)` and `1/x` are irrational and 4 fractional digits accumulate visible rounding error across a deep subtree. The `__count` companion is the standard bigint sum\_count shape, but counts only rows that actually contributed (positive for geometric, non-zero for harmonic), so the display formula uses the right `n`.

The display column reads NULL when the subtree contributed no rows — empty subtree, every contributor filtered out by the positivity / non-zero constraint, or harmonic with a zero reciprocal sum.

## The positivity / non-zero constraint

Both functions have a domain restriction.

- `geometricMean` requires **strictly positive** source values. `LN(0)` is undefined; `LN(negative)` is undefined over the reals.
- `harmonicMean` requires **non-zero** source values. `1 / 0` is undefined.

By default, a save whose source column violates the constraint throws `AggregateSourceConstraintViolationException` — synchronous, with the model class, column name, and offending value in the message. Catch it at the call site or let it propagate.

```php
$segment = new Segment(['growth_rate' => -0.1]);   // throws on save:
$segment->appendToNode($parent)->save();           // AggregateSourceConstraintViolationException
```

Opt into silent-skip semantics with `allowNonPositive()` (fluent form) or `allowNonPositive: true` (attribute form) if your data legitimately includes those values and you want them excluded from the rollup rather than rejected:

```php
#[NestedSetAggregate(
    column: 'growth_geomean',
    geometricMean: 'growth_rate',
    allowNonPositive: true,
)]
class Segment extends Model implements MaintainsTreeAggregates { use NodeTrait; }

// or
Aggregate::geometricMean('growth_rate')->allowNonPositive()->into('growth_geomean');
```

With the modifier, a non-positive (geometric) or zero (harmonic) row contributes nothing to either companion — the SQL fallback `LN(x)` returns NULL for `x ≤ 0` and `NULLIF(x, 0)` handles the harmonic side; the PHP delta-capture matches that. NULL source values are always skipped regardless of the modifier; the constraint only applies to numeric values inside the domain you'd reasonably expect.

The validation runs in the `saving` hook before any delta capture, so a thrown violation aborts the save atomically — no partial drift to clean up.

## How it stays in sync

Every mutation that touches the source column updates the two companions and writes the display value from them in one SET clause:

```text
geometricMean  = EXP(sum_log / NULLIF(count, 0))
harmonicMean   = count / NULLIF(sum_recip, 0)
```

The companion `__count` is transform-aware — it counts only positive (geometric) or non-zero (harmonic) rows, matching the domain of the matching sum. That means `EXP(sum_log / count)` and `count / sum_recip` always use the right `n` even when `allowNonPositive()` is in effect and some rows contribute nothing.

`fixAggregates()` recomputes the companions **and** the display column in one pass — drift detection and repair work identically to `Avg` / `Variance`.

## Empty subtree and single-value semantics

- **Empty subtree** — every variant returns NULL.
- **Single value** — `geometricMean = x`, `harmonicMean = x` (`EXP(LN(x))` and `x/1` both reduce to `x`).
- **All contributors excluded** — display is NULL. With `allowNonPositive()` and every row being non-positive (geometric) or zero (harmonic), the companion `__count` ends at 0 and the formula divides by NULL.

## Numerical precision

The geometric mean is computed via the `EXP(Σ LN(x) / n)` form (the standard for "compute it in SQL once and store it"). For source values that span more than \~13 orders of magnitude, `LN`-then-`EXP` rounds at the last significant fractional digit of `decimal(30, 10)`. For typical workloads — growth percentages, scale factors, ratios within a couple of orders of magnitude — this never matters. If your domain spans much wider ranges, fall back to `withFreshAggregates()` against the native `EXP(AVG(LN(col)))` SQL, which uses the database's full floating-point precision.

The harmonic mean is computed as `count / Σ(1/x)`. Floating-point precision on `decimal(30, 10)` is more than enough for the realistic range — the precision concerns of the geometric mean don't have a direct analog here.

## Filters

Both compose with the standard filter modifiers — `filter`, `filterNotNull`, `filterRaw` — and the package watches the source column for delta maintenance automatically. The same `filterRawWatches` rule applies: list every column the raw SQL references or the stored mean will silently drift.

## Listener form

`GeometricMean` and `HarmonicMean` are also available on `#[NestedSetAggregateListener]` when the per-row contribution is computed in PHP. The companion shape mirrors the SQL form:

```php
use Vusys\NestedSet\Aggregates\AggregateFunction;
use Vusys\NestedSet\Attributes\NestedSetAggregateListener;

#[NestedSetAggregateListener(column: 'score_geomean',  listener: ScoreListener::class, operation: AggregateFunction::GeometricMean)]
#[NestedSetAggregateListener(column: 'score_harmean',  listener: ScoreListener::class, operation: AggregateFunction::HarmonicMean)]
class Monster extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

The migration declares the display column plus the auto-promoted Ln / Recip companions:

```php
$table->decimal('score_geomean', 16, 8)->nullable();
$table->decimal('score_geomean__sum_log', 30, 10)->default(0);
$table->nestedSetAggregate('score_geomean__count');

$table->decimal('score_harmean', 16, 8)->nullable();
$table->decimal('score_harmean__sum_recip', 30, 10)->default(0);
$table->nestedSetAggregate('score_harmean__count');
```

The domain constraint maps onto the listener contract: return `null` from `contribution()` for out-of-domain rows (non-positive for geomean, zero for harmonic) and they are excluded from both the relevant companion sum and the matching count. The display formula then divides by the right `n` and reflects only contributing rows. (Listener mode does not raise `AggregateSourceConstraintViolationException` the way the SQL form does — return `null` upstream if you need fail-loud semantics; otherwise out-of-domain rows are silently skipped.)

`filter:` / `filterNotNull:` parameters compose with the means exactly as in the SQL form.

## Limitations

### `exclusive: true` routes through chain recompute

Same trade-off as exclusive `Avg` / `Variance` — correct but slower than the inline delta. Prefer the default inclusive declarations unless the exclusive value is what the domain actually wants.

### Backend support is uniform

Both functions emit `EXP(LN(...))` / `1.0 / col` against ordinary `SUM` / `COUNT` companions — no native `GEOMETRIC_MEAN` / `HARMONIC_MEAN` SQL functions are required, so MySQL, MariaDB, PostgreSQL, and SQLite all behave identically.
