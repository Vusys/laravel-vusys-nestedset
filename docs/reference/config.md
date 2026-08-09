# Configuration

`config/nestedset.php`:

```php
return [
    'columns' => [
        'lft'       => 'lft',
        'rgt'       => 'rgt',
        'parent_id' => 'parent_id',
        'depth'     => 'depth',
    ],

    'auto_transaction' => true,

    'aggregate_locking' => 'auto',   // 'auto' | 'always' | 'never'

    'queue' => [
        'connection' => env('NESTEDSET_QUEUE_CONNECTION'),
        'queue'      => env('NESTEDSET_QUEUE'),
    ],

    'events_enabled' => true,

    'materialised_path' => [
        'defaults' => [
            'separator'                => '/',
            'wrap'                     => true,
            'maxLength'                => 1024,
            'rejectSeparatorInSegment' => true,
            'uniquePerParent'          => true,
        ],
        'class_defaults' => [],
    ],
];
```

## `columns`

Column names are read globally — change them once in config and every model using `NodeTrait` picks up the new names via the `getLftName()` / `getRgtName()` / `getParentIdName()` / `getDepthName()` accessors.

To use different column names per model, override those accessors on the model:

```php
class Category extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;

    public function getLftName(): string  { return 'tree_lft'; }
    public function getRgtName(): string  { return 'tree_rgt'; }
}
```

## `auto_transaction`

When `true` (default), every tree mutation (`save()` after a `appendToNode` / `prependToNode` / `insertBeforeNode` / `insertAfterNode` / `makeRoot` / `up` / `down`) is wrapped in a `DB::transaction()` so the gap-shift UPDATE, the row INSERT/UPDATE, and any aggregate maintenance hooks all commit together. Set to `false` if you wrap calls in your own transaction at the call site.

## `aggregate_locking` {#aggregate\_locking}

Controls whether the aggregate maintenance path issues `SELECT ... FOR UPDATE` on the ancestor chain before recomputing MIN/MAX (or raw-filter) columns. The right setting for almost every application is `'auto'`.

- **`'auto'`** (default) — lock the ancestor chain only on the recompute path (MIN, MAX, raw-filter, `fixAggregates`). Delta-only updates (SUM, COUNT, AVG) rely on the engine's single-statement row locks, which are sufficient under default isolation on all supported backends.
- **`'always'`** — accepted as a forward-compatible alias for the recompute-path locking. **Today it behaves identically to `'auto'`**: the recompute path (MIN, MAX, raw-filter, `fixAggregates`) locks, and the pure-delta path (SUM/COUNT/AVG) does not. Separate per-delta locking is not yet implemented, so choosing `'always'` does not (yet) add locks beyond `'auto'`.
- **`'never'`** — issue no explicit locks. Marginally faster on the recompute path; can produce drift on PostgreSQL `READ COMMITTED` with concurrent recomputes against overlapping subtrees.

## `queue`

Routing used by `Model::queueFixAggregates()` when the caller doesn't pass an explicit `onConnection:` / `onQueue:` override. Either key may be `null` — that falls back to Laravel's default queue connection / queue name. The defaults pull from environment so you can override per-deployment without code changes:

```env
NESTEDSET_QUEUE_CONNECTION=redis
NESTEDSET_QUEUE=aggregates-low
```

## `events_enabled`

When `true` (default), the package fires typed events on Laravel's event bus around every meaningful operation — see the [Events](events.html) page for the full catalogue, payloads, and recipes.

Set to `false` to short-circuit every firing site. Useful only on genuinely hot paths where you've measured the cost of constructing event objects you'll never observe.

## `materialised_path`

Global defaults for `#[NestedSetMaterialisedPath]` columns. See [Materialised Paths](../tree-operations/materialised-paths.html) for the full feature surface; this section documents only the config knobs.

### Resolution order

For any given path on any given model, the effective value of each knob is resolved most-specific-first:

1. Per-path explicit value (attribute arg or fluent setter on the column).
2. `#[NestedSetMaterialisedPathDefaults(...)]` on the model class (walked through parent classes).
3. `nestedset.materialised_path.class_defaults.<FQCN>` — exact FQCN match, no `is_a` walk.
4. `nestedset.materialised_path.defaults` — global fallback.
5. Package hard-coded fallback (matches the shipped `defaults` block).

### `defaults`

The fallback applied when no more-specific value is set. Defaults match the values shipped in `config/nestedset.php`:

| Key                        | Default | What it does                                                                                                                                                                                                                                                      |
| -------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `separator`                | `'/'`   | Character (or short string) joining segments. Forbidden inside any segment when `rejectSeparatorInSegment` is true.                                                                                                                                               |
| `wrap`                     | `true`  | Wrap the path in leading + trailing separator. `true` → `/a/b/c/`; `false` → `a/b/c`. `LIKE 'a/b/%'` queries work on either, but a wrapped path makes "starts with" tests unambiguous.                                                                            |
| `maxLength`                | `1024`  | Maximum stored path length. Writes that would exceed this throw `PathTooLongException` at save time, before the row is written. Match it to the underlying column width.                                                                                          |
| `rejectSeparatorInSegment` | `true`  | When `true`, a segment that itself contains the separator throws `InvalidPathSegmentException`. Catches a class of silent data corruption (a slug like `a/b/c` would split into three segments on read). Disable only if you really need separators in your data. |
| `uniquePerParent`          | `true`  | When `true`, two siblings of the same parent are not allowed to produce the same segment. Collisions throw `DuplicatePathSegmentException` on save. Disable for paths where collisions are tolerable (key paths can't collide; slug paths usually shouldn't).     |

### `class_defaults`

A map of fully-qualified model class names to per-class default overrides. Useful for layering defaults onto vendor models you can't decorate with a `#[NestedSetMaterialisedPathDefaults]` attribute:

```php
'class_defaults' => [
    \App\Models\Category::class => [
        'separator' => '.',
        'wrap'      => false,
    ],
],
```

Keys are matched **exactly** — there is no `is_a` walk. Subclasses do not inherit a parent's `class_defaults` entry; list each concrete class explicitly. The values are merged onto the global `defaults` block at resolution time, so you only need to override the knobs you want to change.
