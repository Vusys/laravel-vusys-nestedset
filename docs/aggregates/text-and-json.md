# Collection aggregates

Beyond the SQL-standard SUM / COUNT / AVG / MIN / MAX, the package supports four collection aggregates — useful when the question is not "what's the total" but "which distinct values are in this subtree" or "give me a JSON blob of descendant data".

| Kind                              | Stored column type | Empty subtree | Maintenance            |
| --------------------------------- | ------------------ | ------------- | ---------------------- |
| `distinctCount`                   | `bigInteger(0)`    | `0`           | Full subtree recompute |
| `stringAgg`                       | `text` (nullable)  | `NULL`        | Full subtree recompute |
| `stringAgg(...)->distinct()`      | `text` (nullable)  | `NULL`        | Full subtree recompute |
| `jsonAgg` (scalar / list / assoc) | `json` (nullable)  | `NULL`        | Full subtree recompute |
| `jsonObjectAgg`                   | `json` (nullable)  | `NULL`        | Full subtree recompute |

All four are **recompute-only**: every contributing mutation triggers a full subtree recompute over the ancestor chain. There is no delta fast path (the way SUM/COUNT have one) because removing a value from the subtree can't be expressed as a signed delta on the aggregate.

## What the rollups look like

A small product tree with `name` and `tag` columns. The chips on each ancestor show what the maintained `distinct_tags` and `child_names` columns actually hold:

```ns-tree
Catalog {distinct_tags=red, blue, green, child_names=Boots, Cap, Coat, Mittens, Scarf}
  Footwear {distinct_tags=red, blue, child_names=Boots, Cap}
    Boots {tag=red}
    Cap {tag=blue}
  Outerwear {distinct_tags=blue, green, child_names=Coat, Mittens, Scarf}
    Coat {tag=blue}
    Mittens {tag=green}
    Scarf {tag=green}
```

Read the chips as the *contents* of the stored aggregate columns:

- `distinct_tags` on `Outerwear` holds `'blue, green'` — `Mittens` and `Scarf` both have `green` (deduplicated). Adding a row with `tag = 'orange'` under `Outerwear` would trigger a subtree recompute on `Outerwear` and `Catalog`, and the stored value on both would gain `orange` in one UPDATE.
- `child_names` on `Catalog` holds the alphabetised list across the whole subtree. `Catalog.child_names` is a column read — no `GROUP_CONCAT` at query time, no recursive walk.

Every contributing mutation re-runs the subtree aggregate on the ancestor chain. There's no delta path: removing `Mittens` can't subtract from a comma-separated string without scanning what's still there.

## Quick examples

```php
use Vusys\NestedSet\Attributes\NestedSetAggregate;

#[NestedSetAggregate(column: 'distinct_owners', distinctCount: 'owner_id')]
#[NestedSetAggregate(column: 'child_names',     stringAgg: 'name', separator: ', ', limit: 20)]
#[NestedSetAggregate(column: 'distinct_tags',   stringAgg: 'tag', distinct: true)]
#[NestedSetAggregate(column: 'descendant_ids',  jsonAgg: 'id')]
#[NestedSetAggregate(
    column: 'descendant_summary',
    jsonAgg: ['id' => 'id', 'label' => 'name', 'price' => 'price'],
    limit: 50,
)]
#[NestedSetAggregate(
    column: 'slug_to_name',
    jsonObjectAgg: ['key' => 'slug', 'value' => 'name'],
)]
class Category extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

### `distinctCount(source)`

`COUNT(DISTINCT t.{source})` over the subtree. Universal across PG / MySQL / MariaDB / SQLite.

```php
Aggregate::distinctCount('owner_id')->into('distinct_owners');
```

### `stringAgg(source, separator, limit, orderBy)`

Concatenated text aggregate. Default `orderBy` is the source column, so two trees with the same contents produce identical bytes (useful for caching / hashing).

```php
Aggregate::stringAgg('name', separator: ', ', limit: 20)->into('child_names');
```

The `->distinct()` modifier switches to `STRING_AGG(DISTINCT ...)` / `GROUP_CONCAT(DISTINCT ...)`. PG only accepts `ORDER BY` columns that appear in the `DISTINCT` set, so the package enforces this across backends — using `distinct()` together with a custom `orderBy` is rejected at attribute-construction time.

```php
Aggregate::stringAgg('tag')->distinct()->into('distinct_tags');
```

#### Backend caveats

- **SQLite** doesn't honour `ORDER BY` inside `GROUP_CONCAT`. The package emits the same SQL on every backend; on SQLite the output order is backend-defined (typically storage order).
- **SQLite** `GROUP_CONCAT(DISTINCT ...)` doesn't accept a separator argument — the default comma-space separator is used regardless of what you pass.
- **MySQL** has a `group_concat_max_len` server variable. Set it to at least `4 * limit * avg_char_bytes` for production use.

### `jsonAgg($source, limit, orderBy)`

JSON array of values. Three input shapes:

| `$source`                            | Resulting JSON element shape                                  |
| ------------------------------------ | ------------------------------------------------------------- |
| `'id'`                               | Scalar values: `[1, 2, 3]`                                    |
| `['id', 'name']` (list)              | Objects keyed by column name: `[{"id":1,"name":…}, …]`        |
| `['key' => 'id', 'label' => 'name']` | Objects keyed by the array's keys: `[{"key":1,"label":…}, …]` |

The assoc form is the escape hatch when the JSON key needs to differ from the column name (snake\_case → camelCase, renaming for a frontend contract, etc.).

```php
// Declare the cast on the model:
protected $casts = ['descendant_ids' => 'array'];
```

### `jsonObjectAgg(key, value, allowNullKeys)`

Build a JSON object map (`{<key>: <value>, …}`) from descendant rows.

```php
Aggregate::jsonObjectAgg(key: 'slug', value: 'name')->into('slug_to_name');
```

#### Cross-backend key handling

PG's `JSON_OBJECT_AGG` requires string keys and raises on null keys. The package auto-casts integer / UUID / date keys to text and filters null keys via `FILTER (WHERE key IS NOT NULL)` — pass `allowNullKeys: true` to opt out of the null-key filter where the backend supports null keys (MySQL / MariaDB / SQLite). With duplicate keys, **last-row wins** on every supported backend.

## Lexicographic min/max — already supported

The existing `Aggregate::min` / `Aggregate::max` factories work unchanged on text columns and produce lexicographic min/max. Useful for "first alphabetical descendant tag", "last-modified-by name in this subtree", etc.

```php
#[NestedSetAggregate(column: 'first_tag_alpha', min: 'tag')]
```

The `<` / `>` comparison uses the column's declared collation, just like a regular `MIN(text_col)` query.

## Maintenance & drift

All four collection-aggregate kinds maintain themselves through the standard lifecycle hooks — same as MIN / MAX — but always trigger a full subtree recompute on the ancestor chain (no cheap-skip). For very wide subtrees with frequent writes, consider running these in deferred mode:

```php
Category::withDeferredAggregateMaintenance(function (): void {
    // bulk insert / move / delete here
});
// → one fixAggregates() call at the outer-most exit
```

Drift detection is JSON-aware (see [Drift](drift.html)): semantically equal but byte-different JSON values (key reordering after a PG `jsonb` round-trip, whitespace differences) don't report drift. Distinct stringAgg is compared as a sorted set of segments — backend ordering differences don't trip the check.

## Schema setup

The Blueprint macro accepts three new type families:

```php
$table->nestedSetAggregate('distinct_owners',   type: 'distinct_count');
$table->nestedSetAggregate('child_names',       type: 'string_agg');
$table->nestedSetAggregate('descendant_ids',    type: 'json');
$table->nestedSetAggregate('slug_to_name',      type: 'json');
```

- `distinct_count` → `bigInteger($column)->default(0)`
- `string_agg` → `text($column)->nullable()`
- `json` → `json($column)->nullable()` — Laravel routes this to `jsonb` on PG, `json` on MySQL/MariaDB, and `text` on SQLite automatically.
