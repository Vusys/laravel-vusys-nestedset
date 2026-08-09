# Walking Subtrees

The walker is an in-memory traversal helper. You hand it a loaded subtree, you get a generator that yields nodes in DFS pre-order, DFS post-order, or BFS — plus a visitor-form `walk()` that exposes depth, parent, and sibling info per visit.

It is purely a consumer of data you have already loaded. The walker never queries the database.

## What you get

```php
// Pre-order DFS — root first, then each subtree left-to-right.
foreach ($electronics->dfs() as $node) {
    echo $node->name, "\n";
}

// Visitor form with per-visit context.
$electronics->walk(function (Model $node, WalkContext $ctx) {
    echo str_repeat('  ', $ctx->depth), $node->name, "\n";
});

// One-liner for "give me the subtree as a Collection in pre-order".
$ordered = $electronics->flattenedSubtree();
```

Five public methods land on every `NodeTrait` model:

| Method                                              | Returns              | Order                                                    |
| --------------------------------------------------- | -------------------- | -------------------------------------------------------- |
| `dfs($subtree?, $filter?)`                          | `Generator<Model>`   | DFS pre-order                                            |
| `dfsPostOrder($subtree?, $filter?)`                 | `Generator<Model>`   | DFS post-order                                           |
| `bfs($subtree?, $filter?)`                          | `Generator<Model>`   | Breadth-first                                            |
| `walk($visitor, $strategy?, $subtree?, $filter?)`   | `void`               | Visitor-driven; strategy = `'pre'`, `'post'`, or `'bfs'` |
| `flattenedSubtree($strategy?, $subtree?, $filter?)` | `EloquentCollection` | Strategy of your choice                                  |

## The loaded-subtree contract

> [!NOTE]
> The walker never queries the database. Eager-load `descendants` (via `->load('descendants')` or `Category::with('descendants')`) or pass a collection explicitly via the `$subtree` argument. If you have narrowed the subtree on purpose — a filtered eager-load, a depth-limited slice, an ad-hoc collection — the walker honours that scope rather than silently widening it.

When you omit `$subtree`, the walker reads `$this->descendants`. If the relation is not loaded **and** you pass no collection, it throws `UnloadedSubtreeException`:

```php
$electronics->walk($visitor);
// UnloadedSubtreeException: the walker is in-memory only — call
// ->load('descendants') first or pass a collection explicitly
// ($subtree argument). The walker does not query the database.
```

Three load patterns cover almost every case:

```php
// 1. Eager-load up front (one query).
$root = Category::with('descendants')->find($id);
$root->walk($visitor);

// 2. Lazy-load on demand (one extra query).
$root = Category::find($id);
$root->load('descendants');
$root->walk($visitor);

// 3. Explicit, scoped load — the walker honours whatever slice you supply.
$slice = $root->descendants()->where('depth', '<=', 2)->get();
$root->walk($visitor, subtree: $slice);
```

The explicit form is the right tool when you want to walk a *partial* subtree — the walker treats nodes whose children are missing from the slice as leaves and keeps going.

## Strategies side by side

For the tree

```ns-tree
Electronics
  Laptops
  Phones
    iPhone
    Android
```

each strategy yields a different order:

| Strategy                  | Visit order                                   |
| ------------------------- | --------------------------------------------- |
| `'pre'`  (DFS pre-order)  | Electronics, Laptops, Phones, iPhone, Android |
| `'post'` (DFS post-order) | Laptops, iPhone, Android, Phones, Electronics |
| `'bfs'`  (Breadth-first)  | Electronics, Laptops, Phones, iPhone, Android |

(The pre-order and BFS orders happen to match on this shallow tree because every parent has its children grouped under it. On deeper trees they diverge.) The `lft` / `rgt` badges on each row give you the underlying nested-set bounds the walker uses internally — DFS pre-order is the same order as iterating by `lft` ascending, which is why an already-loaded `descendants` relation walks for free.

Pre-order is the default everywhere — it's what humans expect, and what every other tree-printing utility in this package uses.

## The visitor signature

`walk()` calls your closure as

```php
function (Model $node, WalkContext $ctx): WalkSignal|null
```

`WalkContext` carries everything the walker knows about the current visit:

| Field            | Type          | What it means                                                                                                        |
| ---------------- | ------------- | -------------------------------------------------------------------------------------------------------------------- |
| `depth`          | `int`         | Relative to the walk root, **not** the absolute `depth` column on the node. The root itself is depth `0`.            |
| `parent`         | `?Model`      | The hydrated parent from the same subtree, or `null` at the walk root.                                               |
| `siblingIndex`   | `int`         | 0-indexed position among siblings under the same parent in the loaded subtree.                                       |
| `siblingCount`   | `int`         | Total number of siblings under the same parent.                                                                      |
| `isFirstSibling` | `bool`        | Derived; both flags are `true` for an only child.                                                                    |
| `isLastSibling`  | `bool`        | Same.                                                                                                                |
| `pathToRoot()`   | `list<Model>` | Ancestor chain from the current node up to (but excluding) the walk root. Empty at the root. Computed on first call. |

The absolute `depth` column on the node is unchanged — read it via `$node->getDepth()` if you need it.

The ASCII-tree branch-character problem is the canonical motivating example:

```php
$electronics->walk(function (Model $node, WalkContext $ctx) {
    $branch = match (true) {
        $ctx->depth === 0      => '',
        $ctx->isLastSibling     => '└── ',
        default                 => '├── ',
    };
    echo $branch, $node->name, "\n";
});
```

## Signals

Return a `WalkSignal` from the visitor to steer the walk:

| Signal                    | What it does                                                                                                                                                   |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `WalkSignal::SkipSubtree` | Don't descend into this node's children. Honoured by pre-order DFS and BFS; ignored by post-order (children are already visited by the time the parent fires). |
| `WalkSignal::Stop`        | Halt the walk immediately. No further visitors are called.                                                                                                     |
| `null` (or no return)     | Continue normally.                                                                                                                                             |

```php
$electronics->walk(function (Model $node, WalkContext $ctx) {
    if ($node->name === 'Phones') {
        return WalkSignal::SkipSubtree;
    }
    if ($node->discontinued) {
        return WalkSignal::Stop;
    }
});
```

## Pruning with `WalkFilter`

`WalkFilter` is the static-pruning counterpart to the visitor's runtime signals. Use it when the decision about whether to visit a node doesn't depend on what the visitor does — depth ceilings, predicates, "skip everything trashed" rules.

```php
use Vusys\NestedSet\Walker\WalkFilter;

// Root + two levels.
$electronics->walk($visitor, filter: WalkFilter::depth(2));

// Only nodes the predicate accepts. Pruned nodes' subtrees are
// also skipped — there is no way to skip one node but visit its
// children.
$electronics->walk($visitor, filter: WalkFilter::where(
    fn (Model $n, WalkContext $ctx) => $n->active,
));

// AND two filters together: stricter depth wins; predicates compose.
$filter = WalkFilter::compose(
    WalkFilter::depth(3),
    WalkFilter::where(fn (Model $n, WalkContext $ctx) => $n->published),
);

// The instance-form `andThen()` reads better in pipelines.
$filter = WalkFilter::depth(3)->andThen(WalkFilter::where($predicate));
```

The full shape of a filter:

| Field         | Type                                 | Notes                                                                                                          |
| ------------- | ------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| `maxDepth`    | `?int`                               | `null` = unlimited. Inclusive — `depth(2)` allows depths `0, 1, 2`. Relative to the walk root.                 |
| `visitable`   | `?Closure(Model, WalkContext): bool` | `null` = visit everything. Returning `false` skips the node **and** its subtree.                               |
| `includeRoot` | `bool`                               | Default `true`. `false` visits descendants only; depth numbering still treats the (skipped) root as depth `0`. |

Every walker method — `dfs()`, `dfsPostOrder()`, `bfs()`, `walk()`, `flattenedSubtree()` — accepts a `WalkFilter` as its last argument.

> [!TIP]
> The walker's pruning is the same primitive the exporters use internally. If you find yourself reaching for `AsciiOptions::maxDepth`, the underlying `WalkFilter` is what's doing the work — and you can use it directly with `walk()`. All four exporter option objects (`AsciiOptions`, `MermaidOptions`, `DotOptions`, `JsonOptions`) now carry a `?WalkFilter $filter` field so you can apply the same predicate to a Mermaid diagram, an ASCII tree, and a JSON export.

## Generators vs `walk()`

`dfs()`, `dfsPostOrder()`, `bfs()` return PHP generators. `walk()` takes a callback. Both shapes are useful:

> [!TIP]
> Use the generators (`dfs()` and friends) when you want to compose with PHP's iterator pipeline — `foreach` + `break`, `iterator_to_array`, custom map/filter. Use `walk()` when you're doing a side-effecting pass per node and want the per-visit `WalkContext`.

```php
// Generator — break early.
foreach ($electronics->dfs() as $node) {
    if ($node->matches($query)) {
        return $node;
    }
}

// Visitor — context-rich rendering.
$lines = [];
$electronics->walk(function (Model $n, WalkContext $ctx) use (&$lines) {
    $lines[] = str_repeat('  ', $ctx->depth) . $n->name;
});
```

The generators do not support `WalkSignal::SkipSubtree` — they iterate every node the filter admits and let you `break` or `continue` externally. If you need dynamic per-node descent control, use `walk()`.

## `flattenedSubtree()`

A convenience for "give me an ordered Collection" — composition of `walk()` + collect:

```php
$ordered = $electronics->flattenedSubtree('pre');
$ordered = $electronics->flattenedSubtree('bfs', filter: WalkFilter::depth(2));
```

The difference between `flattenedSubtree('pre')` and `descendants()->defaultOrder()->get()` is where the work happens:

- `flattenedSubtree` reads the already-loaded collection and reorders in PHP. No SQL.
- `defaultOrder()->get()` issues a fresh `ORDER BY lft` query against the database.

If you have already loaded the subtree, `flattenedSubtree` is free. If you haven't, do the query.

## Pitfalls

> [!CAUTION]
> Mutating `parent_id` mid-walk produces undefined visit order. The walker's parent → children index is built once at the start; changing the tree structure during traversal can cause nodes to be visited twice, skipped, or appear under the wrong parent. If you need to restructure during traversal, collect the targets in the visitor and apply mutations after the walk finishes.

Other rough edges to know about:

### Orphans

Nodes in the loaded collection whose `parent_id` doesn't match any other loaded row (and isn't the walk root) are unreachable from the root and silently skipped. Walk by reachability, not by raw membership.

### Partial loads are honoured

If you load only depth 0 + 1, the walker treats the depth-1 nodes as leaves. It will not fetch their children for you.

### Attribute mutations are fine

Mutating non-structural attributes (e.g. `$node->name = 'new'`) inside a visitor is safe — the walker doesn't observe attribute changes. The structural index is only sensitive to `parent_id`.

## Comparison with `toTree()`

`NodeCollection::toTree()` (see [In-memory Tree Shaping](tree-shaping.md)) and the walker share the same input shape (a flat collection) but answer different questions.

| Shape                             | Returns                                     | Best for                                                                                  |
| --------------------------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `$collection->toTree()`           | Root model(s) with nested `children` arrays | Blade recursive includes, JSON nested rendering, anything that wants the *data structure* |
| `$root->flattenedSubtree()`       | Collection in chosen order                  | "Give me an ordered list", table rendering                                                |
| `$root->walk($visitor)` / `dfs()` | Visit each node in order; no return         | Side-effecting passes, search, validation                                                 |

Choose the walker when you only need to *visit* each node. Choose `toTree()` when you need the nested arrays to hand to a template.

## `UnloadedSubtreeException`

Thrown by `walk()` / `dfs()` / `dfsPostOrder()` / `bfs()` / `flattenedSubtree()` when no explicit `$subtree` is supplied and the `descendants` relation is not loaded. Resolution: eager-load with `->load('descendants')` or pass a collection explicitly. The walker never issues its own queries — this exception forces the load decision to be visible at the call site so a deliberately narrowed subtree is never accidentally widened.
