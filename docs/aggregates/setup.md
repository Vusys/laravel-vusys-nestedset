# Migration & Setup

## Migration helper

```php
Schema::create('categories', function (Blueprint $table): void {
    $table->id();
    $table->string('name');
    $table->unsignedInteger('articles')->default(0);
    $table->nestedSet();

    // SUM / COUNT — non-null, default 0
    $table->nestedSetAggregate('articles_total');
    $table->nestedSetAggregate('articles_count_all');

    // AVG — nullable decimal; null on empty subtree
    $table->nestedSetAggregate('articles_avg', type: 'avg');

    // MIN / MAX — nullable; empty subtree yields NULL
    $table->nestedSetAggregate('articles_min', type: 'min_max');
    $table->nestedSetAggregate('articles_max', type: 'min_max');
});
```

## `type:` reference

The macro picks the column shape from the `type:` argument. Default is `'sum_count'`. Every type that names a companion-derived family (Avg, Variance, Stddev, WeightedAvg, BoolOr, BoolAnd, GeometricMean, HarmonicMean) **automatically allocates the matching companion columns** alongside the user-facing column — you only ever name the display column in the migration; the companions appear and disappear with the matching macro call.

| `type:` value             | Used for                    | Display column shape           | Nullable / default    | Companions auto-allocated         |
| ------------------------- | --------------------------- | ------------------------------ | --------------------- | --------------------------------- |
| `'sum_count'` *(default)* | `sum`, `count`              | `bigint`                       | NOT NULL, default `0` | —                                 |
| `'avg'`                   | `avg`                       | `decimal(12, 4)`               | nullable              | `*__sum`, `*__count`              |
| `'min_max'`               | `min`, `max`                | `bigint`                       | nullable              | —                                 |
| `'variance'`              | `variance`                  | `decimal(30, 6)`               | nullable              | `*__sum`, `*__sum_sq`, `*__count` |
| `'stddev'`                | `stddev`                    | `decimal(30, 6)`               | nullable              | `*__sum`, `*__sum_sq`, `*__count` |
| `'weighted_avg'`          | `weightedAvg`               | `decimal(12, 4)`               | nullable              | `*__sum_wx`, `*__sum_w`           |
| `'bool_or'`               | `boolOr`                    | `boolean` (backend-native)     | nullable              | `*__sum`, `*__count`              |
| `'bool_and'`              | `boolAnd`                   | `boolean` (backend-native)     | nullable              | `*__sum`, `*__count`              |
| `'geometric_mean'`        | `geometricMean`             | `decimal(12, 4)`               | nullable              | `*__sum_log`, `*__count`          |
| `'harmonic_mean'`         | `harmonicMean`              | `decimal(12, 4)`               | nullable              | `*__sum_recip`, `*__count`        |
| `'bitwise'`               | `bitOr`, `bitAnd`, `bitXor` | `bigint`                       | nullable              | —                                 |
| `'distinct_count'`        | `distinctCount`             | `bigint`                       | NOT NULL, default `0` | —                                 |
| `'string_agg'`            | `stringAgg`                 | `text`                         | nullable              | —                                 |
| `'json'`                  | `jsonAgg`, `jsonObjectAgg`  | `json` / `jsonb` (per backend) | nullable              | —                                 |

There are also three internal companion shapes — `'sum_sq'`, `'decimal_sum'`, `'high_precision_sum'` — that you'll see referenced in error messages but never pass yourself. The macro picks the right one for each companion based on the parent type.

The pattern is consistent: **non-null with `0` default** for the count-shaped kinds (sum, count, distinct\_count) where "no contributors" naturally means zero; **nullable** for every other kind where "no contributors" should be distinguishable from "every contributor returned zero" (matching the SQL convention that `AVG`, `MIN`, `MAX`, `BIT_OR`, etc. on an empty set return `NULL`).

Quantile aggregates (`median`, `percentile`) are read-only via `withFreshAggregates()` and have no `type:` — they cannot be stored as a column. See [Quantiles](quantiles.html).

Listener aggregate columns use the same macro — the type is determined by the operation the listener implements; see [Listener Aggregates](listeners.html) for the per-operation guidance (especially when your `contribution()` returns floats).

## Dropping aggregate columns

`dropNestedSetAggregate('col', type: ...)` drops the display column **and the same companion set the matching `nestedSetAggregate(...)` allocated**. Pass the same `type:` you used when creating the column, or the migration will skip the companions and leave them stranded.

```php
Schema::table('categories', function (Blueprint $table): void {
    $table->dropNestedSetAggregate('articles_avg', type: 'avg');
    $table->dropNestedSetAggregate('articles_total');  // default 'sum_count'
});
```

## Required model conventions

Aggregate columns are derived state. Two rules:

```php
class Category extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;

    // Aggregate columns must NOT appear in $fillable — the package
    // enforces this and throws AggregateConfigurationException at boot
    // if any aggregate column is found there. They stay readable,
    // castable, hideable; only mass-assignment is rejected.
    protected $fillable = ['name', 'articles'];
    // (Equivalently: list every aggregate column in $guarded if you
    // prefer the deny-list style. The package checks $fillable first
    // and falls back to $guarded.)

    // Declare casts manually — NodeTrait does not register them for you.
    protected $casts = [
        'articles'           => 'integer',
        'articles_total'     => 'integer',
        'articles_count_all' => 'integer',
        'articles_avg'       => 'decimal:4',
        'articles_min'       => 'integer',
        'articles_max'       => 'integer',
    ];
}
```

### Scope of the mass-assignment guard

The check fires once at registry build (when the model's aggregate definitions are first resolved) via reflection on the `$fillable` / `$guarded` properties. That covers the common configurations.

It does **not** intercept dynamic bypasses applied at runtime:

- Overriding `isFillable()` or `isGuarded()` on the model to return values not derivable from the properties.
- Calling `Model::unguard()` (e.g. in a service provider, seeder, or test setup) — this is a global Eloquent toggle that disables guarding for every model, so an `Article::create([...])` after `unguard()` can write directly to aggregate columns regardless of what the build-time check saw.

In both cases, the next mutation through the package overwrites the clobbered value with the correctly recomputed aggregate, so the only observable effect is that the value you mass-assigned is silently lost. If you rely on `Model::unguard()` in test code, prefer scoping it with `Model::unguarded(fn () => ...)` and avoid writing to aggregate columns inside the closure.
