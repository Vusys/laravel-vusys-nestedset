# vusys/laravel-nestedset

[![Tests](https://github.com/Vusys/laravel-nestedset/actions/workflows/tests.yml/badge.svg)](https://github.com/Vusys/laravel-nestedset/actions/workflows/tests.yml) [![codecov](https://codecov.io/gh/Vusys/laravel-nestedset/graph/badge.svg)](https://codecov.io/gh/Vusys/laravel-nestedset) [![tests](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/Vusys/laravel-nestedset/badges/tests.json)](https://github.com/Vusys/laravel-nestedset/actions/workflows/tests.yml) [![assertions](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/Vusys/laravel-nestedset/badges/assertions.json)](https://github.com/Vusys/laravel-nestedset/actions/workflows/tests.yml) [![test LOC](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/Vusys/laravel-nestedset/badges/test-ratio.json)](tests/) [![CI matrix](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/Vusys/laravel-nestedset/badges/matrix.json)](.github/workflows/tests.yml) [![Bencher](https://img.shields.io/badge/Bencher-tracked-FD6F1B?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0id2hpdGUiPjxwYXRoIGQ9Ik0xMiAyTDMgN3YxMGw5IDUgOS01VjdaIi8+PC9zdmc+)](https://bencher.dev/perf/vusys-laravel-nestedset) [![Mutation testing](https://img.shields.io/endpoint?style=flat&url=https://badge-api.stryker-mutator.io/github.com/Vusys/laravel-nestedset/master)](https://dashboard.stryker-mutator.io/reports/github.com/Vusys/laravel-nestedset/master) [![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/Vusys/laravel-nestedset/badge)](https://scorecard.dev/viewer/?uri=github.com/Vusys/laravel-nestedset) [![PHP](https://img.shields.io/badge/php-%5E8.3-777BB4?logo=php&logoColor=white)](composer.json) [![Laravel](https://img.shields.io/badge/laravel-11%20%7C%2012%20%7C%2013-FF2D20?logo=laravel)](composer.json) [![PHPStan](https://img.shields.io/badge/PHPStan-level%209-brightgreen.svg)](phpstan.neon) [![Rector](https://img.shields.io/badge/Rector-passing-brightgreen.svg)](rector.php) [![Code Style: Pint](https://img.shields.io/badge/code%20style-Laravel%20Pint-FF2D20.svg?logo=laravel)](https://github.com/laravel/pint) [![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

A modern Laravel implementation of the nested-set model for hierarchical data — strict types throughout, PHPStan level 9, atomic CASE-WHEN mutations, multi-tree scoping, soft-delete cascade, live aggregate roll-ups (eager or lazy with TTL), subtree cloning, JSON tree import/export, materialised-path columns, and an opinionated repair toolkit.

```php
use App\Models\BudgetItem;

$root = new BudgetItem(['name' => 'Engineering']);
$root->saveAsRoot();

$child = new BudgetItem(['name' => 'Salaries', 'cost' => 26000]);
$child->appendToNode($root)->save();

$child->depth;                       // 1
$child->isLeaf();                    // true
$child->isDescendantOf($root);       // true
$child->parent;                      // $root  (Eloquent relation, eager-loadable)
$child->ancestors()->get();          // collection containing $root

$root->refresh();                    // re-read parent bounds after the append
$root->descendants()->get();         // collection containing $child
$root->getDescendantCount();         // 1  (descendants, excluding self; +1 for total nodes in subtree)
```

Declare aggregates on the model and the SUM / COUNT / AVG / MIN / MAX roll-ups are maintained automatically as the tree changes:

```php
use Illuminate\Database\Eloquent\Model;
use Vusys\NestedSet\Attributes\NestedSetAggregate;
use Vusys\NestedSet\Contracts\MaintainsTreeAggregates;
use Vusys\NestedSet\Export\AsciiOptions;
use Vusys\NestedSet\NodeTrait;

#[NestedSetAggregate(column: 'cost_total',      sum:   'cost')]
#[NestedSetAggregate(column: 'item_count',      count: true)]
#[NestedSetAggregate(column: 'avg_cost',        avg:   'cost')]
#[NestedSetAggregate(column: 'biggest_item',    max:   'cost')]
#[NestedSetAggregate(column: 'recurring_total', sum:   'cost', filter: ['recurring' => true])]
class BudgetItem extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;

    protected $fillable = ['name', 'cost', 'recurring'];   // so the mass-assignment below runs
}

// Render the forest with each node's own cost + rolled-up subtree total:
$render = fn () => BudgetItem::toAsciiTreeForest(new AsciiOptions(
    label: fn ($n) => "{$n->name}  (cost = {$n->cost}, total = {$n->cost_total})",
));

echo $render();
// Engineering              (cost = 0,     total = 32000)
// ├── People               (cost = 0,     total = 28000)
// │   ├── Salaries         (cost = 26000, total = 26000)
// │   └── Bonuses          (cost = 2000,  total = 2000)
// └── Tools                (cost = 0,     total = 4000)
//     ├── SaaS             (cost = 2500,  total = 2500)
//     └── Hardware         (cost = 1500,  total = 1500)
// Operations               (cost = 0,     total = 1500)
// └── Office               (cost = 1500,  total = 1500)

BudgetItem::query()->where('name', '=', 'Bonuses')->first()->update(['cost' => 4000]);
BudgetItem::query()->where('name', '=', 'Engineering')->first()->refresh()->cost_total;   // 34000  — every ancestor updates (Bonuses → People → Engineering)

// move the whole Tools subtree (3 nodes) under Operations — one statement
BudgetItem::query()->where('name', '=', 'Tools')->first()
    ->moveTo(BudgetItem::query()->where('name', '=', 'Operations')->first())
    ->save();
echo $render();
// Engineering              (cost = 0,     total = 30000)   ← old parent shrank (Tools left, taking 4000)
// └── People               (cost = 0,     total = 30000)
//     ├── Salaries         (cost = 26000, total = 26000)
//     └── Bonuses          (cost = 4000,  total = 4000)
// Operations               (cost = 0,     total = 5500)    ← new parent grew by the whole moved subtree
// ├── Office               (cost = 1500,  total = 1500)
// └── Tools                (cost = 0,     total = 4000)
//     ├── SaaS             (cost = 2500,  total = 2500)
//     └── Hardware         (cost = 1500,  total = 1500)

BudgetItem::query()->withFreshAggregates()->get();   // ad-hoc correlated recomputation (read-only — see Aggregates → Drift)
```

## Why nested set?

The nested-set encoding stores `lft` and `rgt` integers on every node so any subtree, ancestor chain, or descendant set is a single `BETWEEN` query — no recursive CTEs, no N+1 loops. The price is that mutations (insert / move / delete) have to shift many rows to keep the lft/rgt sequence dense, so it's best suited to **read-heavy hierarchies**: category trees, menu structures, org charts, comment threads.

This package executes every shift as a single `CASE WHEN UPDATE`, so even a subtree move that touches thousands of rows is one round trip.

## Installation

```bash
composer require vusys/laravel-nestedset
```

The service provider auto-registers Blueprint macros and registers a publishable config file:

```bash
php artisan vendor:publish \
    --provider="Vusys\NestedSet\NestedSetServiceProvider" \
    --tag=nestedset-config
```

See the [Installation guide](https://vusys.github.io/laravel-nestedset/getting-started/installation.html) for the rest of the setup (migration macros, model trait, scoped trees).

## Documentation

Full documentation lives at **[https://vusys.github.io/laravel-nestedset/](https://vusys.github.io/laravel-nestedset/)**.

- **Getting Started** — [Introduction](https://vusys.github.io/laravel-nestedset/) · [Installation](https://vusys.github.io/laravel-nestedset/getting-started/installation.html) · [Migration](https://vusys.github.io/laravel-nestedset/getting-started/migration.html) · [Primary Keys](https://vusys.github.io/laravel-nestedset/getting-started/primary-keys.html) · [Model Setup](https://vusys.github.io/laravel-nestedset/getting-started/model-setup.html)
- **Tree Operations** — [Inserting & Moving](https://vusys.github.io/laravel-nestedset/tree-operations/inserting.html) · [Reordering Siblings](https://vusys.github.io/laravel-nestedset/tree-operations/reordering.html) · [Soft Deletes](https://vusys.github.io/laravel-nestedset/tree-operations/soft-deletes.html) · [Bulk Insertion](https://vusys.github.io/laravel-nestedset/tree-operations/bulk-insertion.html) · [Cloning Subtrees](https://vusys.github.io/laravel-nestedset/tree-operations/cloning.html) · [Materialised Paths](https://vusys.github.io/laravel-nestedset/tree-operations/materialised-paths.html)
- **Querying** — [Tree Queries](https://vusys.github.io/laravel-nestedset/querying/queries.html) · [Eloquent Relations](https://vusys.github.io/laravel-nestedset/querying/relations.html) · [In-memory Tree Shaping](https://vusys.github.io/laravel-nestedset/querying/tree-shaping.html) · [Walking Subtrees](https://vusys.github.io/laravel-nestedset/querying/walking.html) · [Tree Exporters & JSON Import](https://vusys.github.io/laravel-nestedset/querying/exporters.html) · [Tree Diff](https://vusys.github.io/laravel-nestedset/querying/tree-diff.html) · [Inspection](https://vusys.github.io/laravel-nestedset/querying/inspection.html) · [Scoped Trees](https://vusys.github.io/laravel-nestedset/querying/scoped-trees.html)
- **Aggregates** — [Overview](https://vusys.github.io/laravel-nestedset/aggregates/overview.html) · [Setup](https://vusys.github.io/laravel-nestedset/aggregates/setup.html) · [Reading](https://vusys.github.io/laravel-nestedset/aggregates/reading.html) · [Declaring](https://vusys.github.io/laravel-nestedset/aggregates/declaring.html) · [Filtered](https://vusys.github.io/laravel-nestedset/aggregates/filtered.html) · [Collection](https://vusys.github.io/laravel-nestedset/aggregates/text-and-json.html) · [Listeners](https://vusys.github.io/laravel-nestedset/aggregates/listeners.html) · [Variance & Stddev](https://vusys.github.io/laravel-nestedset/aggregates/maths.html) · [Weighted Avg & Booleans](https://vusys.github.io/laravel-nestedset/aggregates/weighted-avg-and-booleans.html) · [Means](https://vusys.github.io/laravel-nestedset/aggregates/means.html) · [Quantiles](https://vusys.github.io/laravel-nestedset/aggregates/quantiles.html) · [Top-K](https://vusys.github.io/laravel-nestedset/aggregates/top-k.html) · [Bitwise](https://vusys.github.io/laravel-nestedset/aggregates/bitwise.html) · [Lazy](https://vusys.github.io/laravel-nestedset/aggregates/lazy.html) · [Recipes](https://vusys.github.io/laravel-nestedset/aggregates/recipes.html) · [Maintenance](https://vusys.github.io/laravel-nestedset/aggregates/maintenance.html) · [Drift & Limitations](https://vusys.github.io/laravel-nestedset/aggregates/drift.html)
- **Maintenance** — [Tree Repair](https://vusys.github.io/laravel-nestedset/maintenance/fix-tree.html) · [Repairing Aggregates](https://vusys.github.io/laravel-nestedset/maintenance/fix-aggregates.html) · [Corruption Reference](https://vusys.github.io/laravel-nestedset/maintenance/corruption.html)
- **Reference** — [Configuration](https://vusys.github.io/laravel-nestedset/reference/config.html) · [Testing Helpers](https://vusys.github.io/laravel-nestedset/reference/testing.html) · [Factory Tree Builder](https://vusys.github.io/laravel-nestedset/reference/factories.html) · [Transactions](https://vusys.github.io/laravel-nestedset/reference/transactions.html) · [Events](https://vusys.github.io/laravel-nestedset/reference/events.html) · [Production Notes](https://vusys.github.io/laravel-nestedset/reference/production.html) · [Glossary](https://vusys.github.io/laravel-nestedset/reference/glossary.html)
- **Internals** — [Architecture Overview](https://vusys.github.io/laravel-nestedset/internals/architecture.html)

The site is built from the markdown in [`docs/`](docs/) — if you spot an error, edit the source and open a PR.

## Contributing

Run the full check suite before opening a PR:

```bash
composer pint:check    # style
composer rector:check  # automated refactors
composer analyse       # static analysis
composer test          # unit + feature
```

All four must pass on CI before merge.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for the release history and the pre-1.0 backwards-compatibility breaks called out per version.

## License

MIT. See [LICENSE](LICENSE).
