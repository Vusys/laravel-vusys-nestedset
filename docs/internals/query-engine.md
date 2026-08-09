# Query Engine & Relations

The read side is the payoff of the nested-set encoding: because every node's subtree is the interval `[lft, rgt]`, descendants, ancestors, and siblings are all plain `BETWEEN`/comparison queries — no recursive CTEs, no N+1. This page walks the query builder that exposes those scopes and the custom relations that make them eager-loadable.

## `TreeQueryBuilder` — the tree scopes

Every `NodeTrait` model returns a `TreeQueryBuilder` (`src/Query/TreeQueryBuilder.php`) from `query()`, wired by the `newEloquentBuilder()` override ([Architecture](architecture.html#the-layers-underneath)). It is an Eloquent `Builder` subclass that adds the tree scopes. Each one resolves the configured column name, qualifies it with the table, and appends a predicate over the bounds.

The descendant/ancestor scopes take a `NodeBounds` and translate the containment relation straight into SQL:

```php
public function whereDescendantOf(NodeBounds $bounds, bool $andSelf = false): static
{
    $lft = $this->qualifyColumn($this->lftColumn());
    $rgt = $this->qualifyColumn($this->rgtColumn());

    if ($andSelf) {
        $this->whereBetween($lft, [$bounds->lft, $bounds->rgt]);

        return $this;
    }

    $this->where($lft, '>', $bounds->lft);
    $this->where($rgt, '<', $bounds->rgt);

    return $this;
}
```

The full predicate table:

| Scope                       | SQL predicate                   | Meaning                |
| --------------------------- | ------------------------------- | ---------------------- |
| `whereDescendantOf($b)`     | `lft > b.lft AND rgt < b.rgt`   | Strict descendants.    |
| `whereDescendantOrSelf($b)` | `lft BETWEEN b.lft AND b.rgt`   | Descendants + self.    |
| `whereAncestorOf($b)`       | `lft < b.lft AND rgt > b.rgt`   | Strict ancestors.      |
| `whereAncestorOrSelf($b)`   | `lft <= b.lft AND rgt >= b.rgt` | Ancestors + self.      |
| `whereIsRoot()`             | `parent_id IS NULL`             | Top-level nodes.       |
| `whereIsLeaf()`             | `rgt = lft + 1`                 | No children.           |
| `whereIsAfter($b)`          | `lft > b.rgt`                   | Entirely to the right. |
| `whereIsBefore($b)`         | `rgt < b.lft`                   | Entirely to the left.  |

Plus the ordering/shaping helpers: `defaultOrder()` (`ORDER BY lft ASC` — which is pre-order traversal, i.e. the natural tree order), `reversed()`, `withoutRoot()`, `leaves()`, `root()`, and `withDepth()`. Each mutates the builder in place and returns `static`, so the tree scopes chain with ordinary Eloquent `where`s.

Two of these reach for raw SQL via `TreeExpression` — `whereIsLeaf()` needs a column-to-column comparison (`rgt = lft + 1`) and `withDepth()` aliases the depth column:

```php
public function whereIsLeaf(): static
{
    $lft = $this->qualifyColumn($this->lftColumn());
    $rgt = $this->qualifyColumn($this->rgtColumn());

    $this->whereRaw(new TreeExpression("{$rgt} = {$lft} + 1"));

    return $this;
}
```

### `TreeExpression` — trusted raw SQL

`TreeExpression` (`src/Query/TreeExpression.php`) is a one-line wrapper that implements Laravel's `Expression` contract so a composed SQL string reaches the grammar verbatim:

```php
final readonly class TreeExpression implements ExpressionContract
{
    public function __construct(private string $sql) {}

    public function getValue(Grammar $grammar): string
    {
        return $this->sql;
    }
}
```

Its docblock spells out *why* it exists: Laravel's own `Expression` is typed `@template TValue of literal-string|int|float`, which rules out the dynamically composed (but fully package-owned) SQL the engine emits — column names from config, integer positions, never user input. `TreeExpression` is the escape hatch, used everywhere the package writes a `CASE WHEN` or a column-to-column predicate: the [mutation engine](mutation-engine.html), the leaf/depth scopes here, and the aggregate maintenance SQL.

## Custom relations — eager-loading subtrees

`parent` and `children` are ordinary Eloquent relations keyed on `parent_id` (`src/Concerns/HasTreeRelations.php`). `children()` adds the scope predicates so a multi-tree table doesn't return rows from another tree that happen to share a `parent_id`:

```php
public function children(): HasMany
{
    $relation = $this->hasMany(static::class, $this->getParentIdName());

    foreach (NestedSetScopeResolver::valuesFor($this) as $column => $value) {
        $relation->where($column, '=', $value);
    }

    return $relation;
}
```

`ancestors()` and `descendants()` are different — there is no foreign key for "all my ancestors", so the package ships two custom relation classes (`src/Relations/`) built on the bounds. They share a `BaseRelation` and differ only in the three methods that encode direction. `DescendantsRelation`:

```php
public function addConstraints(): void
{
    if (! self::$constraints) {
        return;
    }

    $query = $this->treeQuery();
    $query->whereDescendantOf($this->parent->getBounds());

    foreach (NestedSetScopeResolver::valuesFor($this->parent) as $col => $value) {
        $query->where($query->qualifyColumn($col), '=', $value);
    }
}
```

Each relation provides three direction-specific seams:

- **`addConstraints()`** — the single-model case: `whereDescendantOf` / `whereAncestorOf` on the parent's bounds.
- **`addEagerConstraint()`** — the eager-load (`with('descendants')`) case. It `orWhere`s one bounds clause per model so a whole set of parents loads in one query: `php $query->orWhere(static function (Builder $q) use (...) { $q->where($lftColumn, '>', $bounds->lft) ->where($rgtColumn, '<', $bounds->rgt); // + scope predicates }); `
- **`matches()`** — the in-PHP step that attaches each loaded row to the right parent, reusing `NodeBounds::contains()`: `php return $model->getBounds()->contains($related->getBounds()); `
- **`relationExistenceCondition()`** — the SQL for `has()` / `whereHas()` subqueries. Descendants: `{hash}.lft between {table}.lft + 1 and {table}.rgt`; ancestors: `{table}.rgt between {hash}.lft and {hash}.rgt and {table}.lft <> {hash}.lft`.

`AncestorsRelation` is the mirror image — `whereAncestorOf`, the inverted `orWhere`, and `contains()` with the arguments swapped (`$related` contains `$model`).

> [!NOTE]
> Every relation and bounds scope carries the scope predicates. The reason is in the source comment: each scope restarts its `lft` sequence at 1, so two trees with overlapping bounds are the *common* case in a multi-tree table. Without the scope filter, `descendants()` would happily return rows from another tree whose bounds fall inside the parent's interval. See [Scoped Trees](../querying/scoped-trees.html).

## Fresh aggregates and the base builder

`withFreshAggregates()` adds correlated-subquery columns that recompute aggregate values at read time (rather than reading the stored rollup) — the drift-detection and ad-hoc-read path. It delegates to `FreshAggregateProjector` (`src/Query/Aggregates/Read/`), which emits one subtree subquery per requested aggregate.

This is also the one place the package needs to influence the *base* query builder. `TreeBaseQueryBuilder` (`src/Query/TreeBaseQueryBuilder.php`) exists solely so the MariaDB fresh-read path can prepend a statement-scoped optimiser hint:

```php
protected function runSelect()
{
    if (! $this->mariaDbSplitMaterializedOff) {
        return parent::runSelect();
    }

    $sql = "SET STATEMENT optimizer_switch='split_materialized=off' FOR "
        .$this->toSql();

    return $this->connection->select($sql, $this->getBindings(), ! $this->useWritePdo);
}
```

MariaDB's planner would otherwise rewrite the derived-table join into a LATERAL DERIVED and lose the materialise-once advantage (\~3× slower). The override is deliberately a single boolean flag consulted in `runSelect()` — the funnel for `get()`/`paginate()`/`chunk()` — so the package can coax the planner for one statement without mutating session state. This backend-specific shaping is part of the same family as `TreeExpression`'s dialect handling: the bounds queries are portable, but the *fast* form of a subtree subquery differs per backend (LATERAL on PostgreSQL/MySQL, derived-table on MariaDB, correlated fallback on SQLite).

## Where to go next

[Aggregate Maintenance](aggregate-maintenance.html) covers the write side of those rollup columns — how they stay correct as the tree changes.
