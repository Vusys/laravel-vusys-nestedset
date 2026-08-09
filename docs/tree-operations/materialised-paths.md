# Materialised Paths

A materialised path is a denormalised string column on a nested-set row that carries the row's ancestry as a separator-joined sequence — `/electronics/laptops/ultrabooks/` for URLs, `Electronics > Laptops > Ultrabooks` for breadcrumbs. The package keeps the column coherent with the tree on every mutation: insert, update, move, rename, bulk insert, subtree clone, tree-diff apply, soft-delete restore, and `fixTree`. Reorder leaves paths untouched by construction (siblings keep the same parent + segment). Querying remains pure Eloquent — the path is just a column.

## Quick start

Declare a column on the model with `#[NestedSetMaterialisedPath]` and add the column to your migration:

```php
use Vusys\NestedSet\Attributes\NestedSetMaterialisedPath;

#[NestedSetMaterialisedPath(column: 'url_path', slug: 'name')]
final class Category extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;
}
```

```php
Schema::create('categories', function (Blueprint $t) {
    $t->id();
    $t->string('name');
    $t->nestedSet();
    $t->string('url_path', 1024)->nullable()->index();
});
```

That's the whole setup. After `$category->save()` the column reflects the row's ancestry:

```php
$node->url_path;                                       // '/electronics/laptops/ultrabooks/'
explode('/', trim($node->url_path, '/'));              // ['electronics', 'laptops', 'ultrabooks']
```

## Sources

Each declaration names exactly one source for the per-row segment. Pick the one that matches what you're storing.

### `slug`

`slug: 'name'` runs `Str::slug($node->name)` per row. Use for URL paths derived from a human-facing display attribute.

```php
#[NestedSetMaterialisedPath(column: 'url_path', slug: 'name')]
```

### `attribute`

`attribute: 'display_name'` reads the column raw, no slugification. Use for breadcrumbs where the rendered text matters (`'Electronics > Laptops & Notebooks'`).

```php
#[NestedSetMaterialisedPath(column: 'crumb_path', attribute: 'display_name', separator: ' > ', wrap: false)]
```

### `key`

`key: true` uses the row's primary key as the segment (`'.1.2.3.'` for an ancestor chain of ids 1→2→3). Stable, never renamed, but unreadable.

```php
#[NestedSetMaterialisedPath(column: 'id_path', key: true, separator: '.')]
```

Autoincrement keys aren't known until the INSERT lands, so key-dependent paths are written by a second targeted UPDATE in the `saved` listener. UUID keys are generated client-side and write inline.

### Closure (method form)

Attributes can't carry closures. For runtime-composed segments use the method form:

```php
use Vusys\NestedSet\MaterialisedPath\MaterialisedPath;

class Article extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;

    protected static function materialisedPaths(): array
    {
        return [
            'breadcrumb_path' => MaterialisedPath::from(
                static fn (self $node): string => Str::slug($node->title, '-'),
            )->separator('/')->maxLength(2048),
        ];
    }
}
```

The method merges on top of the attribute list; a column appearing in both → method wins. The default implementation in the trait returns `[]` so attribute-only models pay nothing.

## Multiple paths

A single model may carry several path columns at once, each derived from a different source attribute or formatted differently. Each column is maintained independently — a rename touching one source attribute writes only the columns whose value actually changes.

```php
#[NestedSetMaterialisedPath(column: 'url_path',   slug: 'name')]
#[NestedSetMaterialisedPath(column: 'crumb_path', attribute: 'display_name', separator: ' > ', wrap: false)]
final class Category extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

A change to `display_name` rewrites `crumb_path` for the row + every descendant; `url_path` stays put.

## Customising defaults

The package resolves a column's effective options through five layers, most specific wins:

1. **Per-path explicit value** — attribute arg or fluent setter on the value object.
2. **`#[NestedSetMaterialisedPathDefaults]`** on the model class (walked through parent classes).
3. **`config('nestedset.materialised_path.class_defaults.'.$class)`** — exact FQCN match, no `is_a` walk.
4. **`config('nestedset.materialised_path.defaults')`** — global fallback.
5. Package hard-coded fallback: `separator: '/'`, `wrap: true`, `maxLength: 1024`, `rejectSeparatorInSegment: true`, `uniquePerParent: true`.

Per-path on the attribute:

```php
#[NestedSetMaterialisedPath(column: 'url_path', slug: 'name', separator: '.', wrap: false)]
```

Class-level:

```php
#[NestedSetMaterialisedPathDefaults(separator: '.', wrap: false, maxLength: 2048)]
#[NestedSetMaterialisedPath(column: 'numeric_path', key: true)]
#[NestedSetMaterialisedPath(column: 'doc_path', slug: 'reference')]
class DocumentNode extends Model implements MaintainsTreeAggregates { use NodeTrait; }
```

Per-class config:

```php
// config/nestedset.php
'materialised_path' => [
    'defaults' => ['separator' => '/', 'wrap' => true, 'maxLength' => 1024],
    'class_defaults' => [
        \App\Models\Category::class => ['separator' => '.', 'wrap' => false],
    ],
],
```

`class_defaults` keys are **exact FQCN** with no inheritance walk — if you want different defaults for two subclasses, list both. Strong use case: overriding a vendor model whose class you can't decorate.

## Reading paths

The path is a column. Read it like any other column. The only read-side affordance the package provides is `materialisedPathFor()` which returns the resolved `MaterialisedPath` value object — useful when a caller needs the separator without parsing it back out of the string.

```php
$node->url_path;                                       // '/electronics/laptops/'
$node->materialisedPathFor('url_path')->getSeparator(); // '/'
```

No `formatPath()`, no Stringable wrapper, no breadcrumb helper — `explode` handles it.

## Validation and exceptions

| Condition                                              | Exception                                | Default                                                          |
| ------------------------------------------------------ | ---------------------------------------- | ---------------------------------------------------------------- |
| Segment is the empty string                            | `EmptyPathSegmentException`              | Throw                                                            |
| Segment contains the separator                         | `InvalidPathSegmentException`            | Throw, unless `rejectSeparatorInSegment(false)` → silently strip |
| Two siblings produce the same segment under one parent | `DuplicatePathSegmentException`          | Throw, unless `uniquePerParent(false)`                           |
| Computed path exceeds `maxLength`                      | `PathTooLongException`                   | Throw                                                            |
| Builder is non-deterministic (dev only)                | `NonDeterministicPathSegmentException`   | Throw when `APP_DEBUG=true`                                      |
| Attribute declares zero or multiple sources            | `MaterialisedPathConfigurationException` | Throw at boot                                                    |

Per-parent uniqueness comparison is byte-exact (`strcmp`). For case-insensitive collision detection, lowercase inside the segment builder itself (`MaterialisedPath::from(fn ($n) => Str::lower(Str::slug($n->name)))`). No comparator config knob — collation semantics are too varied to wrap.

## Repair

`fixMaterialisedPaths()` walks the tree by parent\_id and rebuilds every declared column for every row. Useful when the structural tree is consistent but a path column has drifted — manual SQL edits, pre-feature backfill rows, or a bulk job run inside `withoutMaterialisedPathMaintenance()`.

```php
Category::fixMaterialisedPaths();                      // every declared column
Category::fixMaterialisedPaths('url_path');            // just one column
Category::fixMaterialisedPaths(anchor: $rootCategory); // limit to a subtree
```

`fixTree()` calls this as its final step, so structural repair and path repair always run together. The result's `materialisedPathsRepaired` field is a `column => row-count` map; cross-link the corruption reference for the full taxonomy.

## Bypassing maintenance

For bulk renames where running the listener N times is wasteful:

```php
Category::withoutMaterialisedPathMaintenance(function (): void {
    foreach (Category::cursor() as $node) {
        $node->name = strtolower((string) $node->name);
        $node->save();
    }
});

Category::fixMaterialisedPaths();
```

The bypass counter is reentrant. Wrapping wrappers compose. No async-by-default job ships — the supported pattern for very large bulk renames is the bypass + a follow-up `fixMaterialisedPaths()`, which the user can dispatch to a queue themselves.

## Performance

| Operation                                | With N declared paths                                                       |
| ---------------------------------------- | --------------------------------------------------------------------------- |
| Insert leaf, no key-dependent paths      | unchanged — paths set inline at INSERT                                      |
| Insert leaf, K key-dependent paths       | + K UPDATEs (one per key-dep column)                                        |
| Move leaf                                | + up to N UPDATEs, leaf subtree is one row → cheap; unchanged paths skipped |
| Move subtree                             | + up to N UPDATEs, each bounded by subtree size; unchanged paths skipped    |
| Rename source attribute touching M paths | + M batched UPDATEs (one per column, subtree-bounded)                       |
| Sibling reorder                          | unchanged — no path columns touched                                         |
| Subtree clone                            | + path generation inside the existing clone transaction                     |
| `fixTree()`                              | + (declared paths × N) recomputes, batched per column                       |
| Breadcrumb fetch                         | 0 DB hits — `explode` the column                                            |

One extra UPDATE per changed path per mutation, all bounded by subtree size, all single-statement.

## Limitations

The following are out-of-scope by design — they're not "coming soon":

- **Query helpers**: no `whereDescendantOfPath`, `wherePathEquals`. Write `where('url_path', 'like', $prefix.'%')` in your own scopes; Eloquent handles it.
- **`formatPath()` / Stringable wrapper / breadcrumb helper**: the column is a string; `explode` works.
- **Blueprint macro extension**: column names, lengths, and indexability are migration decisions.
- **Automatic indexing**: index strategy and prefix length depend on backend / query shape.
- **Async-by-default maintenance**: use `withoutMaterialisedPathMaintenance()` + `fixMaterialisedPaths()`.
- **Events on path changes**: existing `saving`/`saved` events fire naturally; subscribers compare `getOriginal($col)` to `$col` if they care.
