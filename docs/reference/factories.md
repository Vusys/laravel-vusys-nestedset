# Factory Tree Builder

The `BuildsNestedSetTrees` trait turns a Laravel `Factory` into a tree-shaped fixture builder. One call hands back a real placed tree, backed by [`bulkInsertTree()`](../tree-operations/bulk-insertion.md) so even depth-3 branching-5 (156 nodes) costs three statements — `makeGap`, bulk write, deferred `fixAggregates` — instead of one hundred fifty-six per-row `appendToNode()->save()` calls.

```php
use App\Models\Category;

// Uniform tree: 1 root + 2 + 4 + 8 = 15 nodes.
$root = Category::factory()->tree(depth: 3, branching: 2)->create();

// Forest with explicit attributes.
$roots = Category::factory()->treeFromShape([
    ['name' => 'Electronics', 'children' => [
        ['name' => 'Laptops', 'children' => [
            ['name' => 'MacBook'],
            ['name' => 'ThinkPad'],
        ]],
        ['name' => 'Phones'],
    ]],
    ['name' => 'Books'],
])->create();
```

## Setup

Add the trait to any factory whose model uses `NodeTrait`:

```php
use Illuminate\Database\Eloquent\Factories\Factory;
use Vusys\NestedSet\Testing\BuildsNestedSetTrees;

/**
 * @extends Factory<Category>
 */
final class CategoryFactory extends Factory
{
    /** @use BuildsNestedSetTrees<Category> */
    use BuildsNestedSetTrees;

    protected $model = Category::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->words(2, true),
        ];
    }
}
```

The trait expects nothing of `definition()` beyond what the factory already returns — it injects tree columns (`lft`, `rgt`, `depth`, `parent_id`) at insert time and strips them from the resolved attribute payload if `definition()` happens to set them. If the model has a `name` column the trait writes a default `Depth d Sibling i` label there; if not, point at another column with `labelColumn:` or pass `labelColumn: null` to opt out (see [Labels](#labels)).

## Uniform trees with `tree()`

```php
public function tree(
    int $depth,
    int|array|Closure $branching,
    ?HasNestedSet $parent = null,
    ?string $labelColumn = 'name',
    ?Closure $per = null,
    bool $afterCreating = true,
): static;
```

`$depth` is the number of generations below the root — `depth: 0` produces a single root, `depth: 3` produces a four-generation tree. `$branching` controls how many children each non-leaf gets, and accepts three shapes:

| Form                                | Meaning                                                                                                        |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `int`                               | Every non-leaf has this many children. Uniform tree.                                                           |
| `list<int>` indexed by parent depth | `[5, 2, 1]` means root has 5 children, each child has 2 grandchildren, each grandchild has 1 great-grandchild. |
| `Closure(int $parentDepth): int`    | Most general — returns the child count for a parent at the given depth.                                        |

### Shape recipes

```php
// Single root (no descendants).
Category::factory()->tree(depth: 0, branching: 0)->create();

// Star — one root, N siblings.
Category::factory()->tree(depth: 1, branching: 50)->create();

// Path / spine — single chain.
Category::factory()->tree(depth: 10, branching: 1)->create();

// Balanced binary.
Category::factory()->tree(depth: 5, branching: 2)->create();

// Pyramid — wide at the root, narrowing below.
Category::factory()->tree(depth: 3, branching: [10, 5, 2])->create();

// Inverted pyramid — narrow at the root, widening below.
Category::factory()->tree(depth: 3, branching: [1, 3, 10])->create();

// Custom shape by depth.
Category::factory()->tree(depth: 4, branching: fn (int $d) => 1 << $d)->create();
```

Asymmetric or skewed trees (where one branch is deeper than its siblings) need [`treeFromShape()`](#explicit-shapes-with-treefromshape) — the uniform variant produces structurally identical subtrees per generation.

### Validation

| Input                                                            | Behaviour                                                                  |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `depth < 0`                                                      | `InvalidArgumentException`                                                 |
| `branching` int `< 1` with `depth > 0`                           | `InvalidArgumentException` (a non-leaf with zero children cannot exist)    |
| `branching` array shorter than `depth`                           | `InvalidArgumentException` — the message names the expected minimum length |
| `branching` array entry `< 1` with deeper levels still requested | `InvalidArgumentException`                                                 |
| `branching` closure returning non-int                            | `InvalidArgumentException`                                                 |
| `branching` closure returning `< 0`                              | `InvalidArgumentException`                                                 |
| `branching` array longer than `depth`                            | Surplus entries silently ignored                                           |

## Explicit shapes with `treeFromShape()`

When you need per-node control — hand-picked labels, asymmetric branches, specific scope-column values, or aggregate source columns — use the nested-array form:

```php
public function treeFromShape(
    array $shape,
    ?HasNestedSet $parent = null,
    bool $afterCreating = true,
): static;
```

Each entry is an associative array of model attributes plus an optional `children` key. Top-level entries become roots (multiple top-level entries = a forest); passing `$parent` makes them direct children of that parent instead. Attribute precedence is `definition() → state(...) → explicit shape attributes → per-row closure` — explicit attributes override factory state for collisions, and missing keys fall through to the factory's defaults.

```php
$roots = Category::factory()->treeFromShape([
    ['name' => 'Electronics', 'children' => [
        ['name' => 'Laptops'],
        ['name' => 'Phones',  'children' => [
            ['name' => 'iPhone'],
            ['name' => 'Pixel'],
        ]],
    ]],
    ['name' => 'Books'],  // sibling root in the same forest
])->create();
```

### `children` is reserved

The `children` key always names structural children — it cannot also be a model attribute. A model with a real `children` column (rare) can't have it populated through `treeFromShape`; use the [per-row closure](#per-row-customisation-with-the-per-closure) or `->state()` instead.

### Empty rows are fine

`treeFromShape([[]])` is a single root whose attributes come entirely from `definition()`. `treeFromShape([['children' => [...]]])` is a root with no explicit attributes but explicit children. Neither requires you to spell out every column.

## Labels

The uniform variant writes a default `Depth d Sibling i` label so rows are distinguishable in test failures. Three knobs cover the common cases:

```php
// Default — labels go to the 'name' column.
Category::factory()->tree(depth: 1, branching: 2)->create();
// Rows: name="Depth 0 Sibling 0", "Depth 1 Sibling 0", "Depth 1 Sibling 1"

// Write labels to a different column.
Category::factory()->tree(depth: 1, branching: 2, labelColumn: 'title')->create();

// Skip label injection entirely; defer to definition().
Category::factory()->tree(depth: 1, branching: 2, labelColumn: null)->create();
```

If `labelColumn` is set to a column that doesn't exist on the model, the trait throws `InvalidArgumentException` upfront with a schema check rather than letting the DB constraint error through. The label is injected after `definition()` and `state()`, so blanket `state(['name' => 'X'])` or explicit shape attributes still override it.

`treeFromShape()` doesn't inject labels — you supply per-node attributes directly.

## Per-row customisation with the `$per` closure

When attributes depend on tree position — depth-keyed weights, parent-inherited tenant IDs, "every Nth sibling" flags, aggregate source columns — pass a closure:

```php
Category::factory()
    ->tree(
        depth: 3,
        branching: 2,
        per: fn (int $depth, int $siblingIndex, ?array $parentAttrs) => [
            'weight' => 10 - $depth,
            'is_promoted' => $siblingIndex === 0,
            'domain' => $parentAttrs['domain'] ?? 'root-domain',
        ],
    )
    ->create();
```

The closure runs once per generated row, after `definition()` / `state()` / explicit shape, and merges last — meaning the closure wins over everything else. It receives:

- **`int $depth`** — 0 for the root, 1 for direct children, and so on.
- **`int $siblingIndex`** — 0-indexed position among siblings under the same parent. Resets per `count()` iteration (see [Forests and `count()`](#forests-and-count)).
- **`?array $parentAttrs`** — the parent row's fully-resolved attributes, `null` for roots.

### `parentAttrs` does not include the primary key

At closure-call time the rows haven't been inserted yet, so no `id` / UUID is available to the closure. If a child needs to reference a parent's key, set the relationship up in an `afterCreating` callback (which fires after rows are persisted), or pre-generate IDs in `definition()` / `treeFromShape` and write them explicitly.

### Spooky randomness

A closure that reads `$parentAttrs['some_random_field']` only re-derives correctly when the parent's randomness is preserved across runs. Seed faker before the build (`fake()->seed(123)`) to keep test output stable.

## Composing with factory state

### `->state()`

State applied to the factory composes normally — it runs for every generated row, exactly like a flat `Category::factory()->count(15)->create()` would:

```php
Category::factory()
    ->state(['type' => 'category'])
    ->tree(depth: 3, branching: 2)
    ->create();
```

State runs through `definition()`, so randomised values vary per row. Per-row overrides that depend on tree position go through the `$per` closure or `treeFromShape` instead.

### `Sequence`

```php
Category::factory()
    ->state(new Sequence(['title' => 'A'], ['title' => 'B'], ['title' => 'C']))
    ->tree(depth: 1, branching: 2)
    ->create();
// title="A" for root, "B" for first child, "C" for second child.
```

Sequences cycle in DFS pre-order — the same order rows are saved to the database. Wraps around if the tree has more rows than sequence entries.

### `afterCreating`

```php
Category::factory()
    ->afterCreating(fn (Category $row) => Log::info("created {$row->id}"))
    ->tree(depth: 1, branching: 2)
    ->create();
```

`afterCreating` callbacks fire once per generated row in DFS pre-order — matching the contract callers expect from a normal factory call. The trait fires them manually because `bulkInsertTree` bypasses Eloquent's per-row `created` event (the bulk path is intentional). For million-row seeders where the per-row hydration cost dominates, opt out:

```php
Category::factory()
    ->tree(depth: 4, branching: 10, afterCreating: false)
    ->create();
```

### Attribute precedence summary

In order, each layer overrides the previous one:

1. `definition()` — factory defaults.
2. Label injection — writes `Depth d Sibling i` to `labelColumn` (uniform `tree()` only; skipped when `labelColumn: null`).
3. `->state(...)` chain — blanket overrides applied to every row.
4. Explicit shape attributes — `treeFromShape()` per-node values.
5. `$per` closure — position-aware per-row overrides.

The per-row closure wins last because it's the most specific layer; blanket `state(['name' => 'X'])` over a per-closure that returns `['name' => 'Y']` produces `'Y'`.

## Forests and `count()`

`count(N)->tree(...)` produces N independent trees, each with its own `bulkInsertTree` call:

```php
$roots = Category::factory()->count(3)->tree(depth: 2, branching: 2)->create();
// 3 roots, each with a 7-node subtree → 21 rows total, 3 bulk insert statements.
```

`$siblingIndex` resets per `count()` iteration — three trees of `tree(depth: 1, branching: 2)` produce three sets of `(0, 1)`, not a global `0..5` sequence.

For multiple roots in **one** bulk-insert statement (one transaction, one gap-open, one aggregate recompute), use `treeFromShape` with multiple top-level entries instead. That's the right shape when the roots logically belong together — a catalog with several top-level categories, for example.

### Ordering: `count()` must come first

`tree(...)->count(N)` is ambiguous — should N multiply trees or rows inside a tree? — and throws `LogicException`. Always put `count()` before `tree()` / `treeFromShape()`.

## Return shape

Following Laravel's factory convention:

- **One root requested** (uniform `tree()`, or `treeFromShape()` with a single top-level entry, with `count()` unset or `1`) → returns the single Model.
- **Multiple roots** (`treeFromShape()` with multiple top-level entries, or any `count(N)` call) → returns an Eloquent `Collection` of the top-level roots.

Descendants aren't auto-loaded on the returned models — the factory writes the tree but doesn't pull whole subtrees into memory by default. Use the package's `descendants` relation (`$root->descendants`) when you need them.

## Scoped models

Scoped models (those with `#[NestedSetScope]` or `getScopeAttributes()`) inherit the scope-column requirement that `bulkInsertTree()` enforces — pass an anchor that supplies the scope:

```php
$anchor = new MenuItem(['name' => 'Anchor', 'menu_id' => 7]);
$anchor->saveAsRoot();
$anchor->refresh();

MenuItem::factory()
    ->state(['menu_id' => 7])
    ->tree(depth: 2, branching: 2, parent: $anchor)
    ->create();
```

If the factory's resolved scope state disagrees with the anchor's scope, the trait throws `ScopeViolationException` upfront with both scopes named — diagnosed before the bulk insert opens its gap. Without scope state or an anchor, a scoped factory falls through to `bulkInsertTree`, which surfaces the same exception.

## Grafting onto an existing parent

Passing `parent:` to either entry point makes the top-level entries direct children of that parent instead of new roots:

```php
$root = Category::query()->whereNull('parent_id')->first();

Category::factory()->tree(depth: 2, branching: 3, parent: $root)->create();
```

Two upfront checks reject error-prone parents:

- **Trashed parent** (`SoftDeletes` model with `deleted_at` set) → `InvalidArgumentException` with a "restore the parent or pass null to seed a fresh root" message. Avoids the opaque `FOREIGN KEY constraint failed` that the DB layer would otherwise raise.
- **Cross-scope parent** (scoped model where factory state's scope differs from parent's scope) → `ScopeViolationException` upfront with both scopes named, before any gap is opened.

## Aggregate fixtures

A factory for a model with aggregate columns ([`#[NestedSetAggregate]`](../aggregates/declaring.md)) supplies the **source** columns in `definition()` or via the `$per` closure; the aggregate columns are derived by the deferred `fixAggregates()` pass that runs at the end of `bulkInsertTree`:

```php
$root = Area::factory()
    ->tree(
        depth: 2,
        branching: 2,
        per: fn (int $d, int $i) => ['tickets' => 1],
    )
    ->create();

$root->refresh();
$root->tickets_total;  // sum across all descendants, computed by the recompute pass.
```

If a `treeFromShape` entry supplies an aggregate column directly (e.g. `tickets_total`), the recompute overwrites it — aggregate columns are derived, not seeded. A test that asserts "recompute matches manual computation" should compute the expected value in PHP and assert against the recomputed column, not seed both.

## Determinism / faker seeding

For reproducible test runs, seed faker before each build:

```php
fake()->seed(123);
$root = Category::factory()->tree(depth: 3, branching: 2)->create();
```

The trait does not manage faker state itself. Without a seed, `definition()` produces fresh values per row (which is usually what tests want — distinct names, distinct timestamps) but two runs of the same test diverge.

## Preview without inserting

`previewTree()` returns the fully-resolved nested-array payload that *would* have been handed to `bulkInsertTree`, without persisting anything:

```php
$payload = Category::factory()->tree(depth: 2, branching: 3)->previewTree();

// $payload is the same shape treeFromShape() accepts as input.
$root = Category::factory()->treeFromShape($payload)->create();
```

Useful for snapshot-testing the planned structure, debugging per-row closures without insertion noise, or pre-generating a deterministic shape that's replayed across multiple tests. The output applies `definition()`, state, explicit attributes, and the `$per` closure — exactly what `create()` would have written.

## `make()` is unsupported

Calling `make()` on a tree shape throws `LogicException` — the "unpersisted tree" representation is ambiguous (no IDs yet for `parent_id`, no `lft`/`rgt` computed, no relations to hydrate). For the no-DB ergonomic case, use `previewTree()` instead, which returns the resolved attribute payload that `treeFromShape` accepts.

## Using in seeders

The same trait works in database seeders. Two things to watch on million-row seeds:

```php
// Batch by top-level root so each bulkInsertTree call's payload stays bounded.
Category::factory()
    ->count(100)
    ->tree(depth: 4, branching: 10, afterCreating: false)
    ->create();
// 100 trees × 11,111 rows each = 1.1M rows, 100 bulk-insert statements,
// each holding only its own 11k-row payload in PHP at a time.
```

1. **Memory.** The shape resolution loads N rows in PHP per `bulkInsertTree` call. At N=1000 it's fine; at N=1M you batch by top-level root as above.
2. **Per-row `afterCreating` cost.** The trait fires `afterCreating` callbacks per row in DFS order — useful for tests, expensive on seeds. Pass `afterCreating: false` to skip them.

### `definition()` side effects

A factory whose `definition()` calls another factory (e.g. `'category_id' => Category::factory()`) creates one extra row per call. With `tree(depth: 3, branching: 5)` that's 156 extra inserts buried inside `definition()`, completely outside the trait's three-statement budget. Either accept the cost, or resolve referenced IDs once at the top of the test and pass them through state:

```php
$categoryId = Category::factory()->createOne()->id;

Tag::factory()
    ->state(['category_id' => $categoryId])
    ->tree(depth: 3, branching: 5)
    ->create();
```

## Common errors

| Condition                                           | Exception                  | What to do                                                                                  |
| --------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------- |
| `depth < 0`                                         | `InvalidArgumentException` | Use `depth: 0` for a single root.                                                           |
| `branching: 0` with `depth > 0`                     | `InvalidArgumentException` | Use `depth: 0` if you want no children.                                                     |
| `branching` array shorter than `depth`              | `InvalidArgumentException` | Provide one entry per generation.                                                           |
| `branching` closure returns non-int or `< 0`        | `InvalidArgumentException` | Return a non-negative int.                                                                  |
| `treeFromShape([])`                                 | `InvalidArgumentException` | Pass at least one top-level entry.                                                          |
| `treeFromShape` entry has non-array `children`      | `InvalidArgumentException` | `children` must be an array of further nodes.                                               |
| `labelColumn:` set to a column that doesn't exist   | `InvalidArgumentException` | The message lists the model's actual columns.                                               |
| `parent:` is soft-deleted                           | `InvalidArgumentException` | Restore the parent (`->restore()`) or pass `null`.                                          |
| `parent:` scope ≠ factory state's scope             | `ScopeViolationException`  | Match the scope in `->state(...)` or pick a different parent.                               |
| Scoped factory with no scope state or anchor        | `ScopeViolationException`  | Pass `parent:` (the trait copies scope from the anchor) or `->state(['scope_col' => ...])`. |
| `$per` closure returns non-array                    | `InvalidArgumentException` | Return an associative array (possibly empty).                                               |
| `tree(...)->make()` or `treeFromShape(...)->make()` | `LogicException`           | Use `create()`, or `previewTree()` for the no-DB case.                                      |
| `tree(...)->count(N)`                               | `LogicException`           | Put `count()` first: `count(N)->tree(...)`.                                                 |

All package-specific exceptions implement the `Vusys\NestedSet\Exceptions\NestedSetException` marker interface (which extends `Throwable`), so you can catch any package-originated failure with a single `catch (\Vusys\NestedSet\Exceptions\NestedSetException $e)` regardless of the SPL base class each one extends.
