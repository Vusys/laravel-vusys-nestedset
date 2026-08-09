# Cloning Subtrees

Duplicate a node and every descendant under a new parent — or as a new root — with fresh primary keys, regenerated structural columns (`lft` / `rgt` / `depth` / `parent_id`), and one deferred aggregate recompute. The whole clone runs in a single transaction; on failure nothing is committed.

`cloneSubtreeTo()` and `cloneSubtreeAsRoot()` are built on top of [`bulkInsertTree`](bulk-insertion.html) — they inherit its autoincrement / UUID parent-id reconciliation and deferred aggregate maintenance, and suppress per-row Eloquent `creating` / `created` / `saving` / `saved` events for the cloned rows. The single signal listeners hook is [`SubtreeCloned`](../reference/events.html#subtree-cloning).

## What it looks like

Two trees side by side make the structural story concrete. Starting state — a re-usable `Template` site lives alongside an empty `Customer 42` site:

```ns-tree
Template
  Home
  About
    Team
    Mission
  Contact
Customer 42
```

A single call clones the whole `Template` subtree under `Customer 42`:

```php
$template->cloneSubtreeTo($customer42);
```

After the call, `Template` is untouched — the clone is independent — and `Customer 42` now carries a fresh copy of every descendant. Note the bound badges on each row (`lft` and `rgt`): the cloned rows occupy a brand-new slot range under `Customer 42`, and `Template`'s own bounds shift outward only if siblings need to make room (they don't, here — `Template` and `Customer 42` are roots and don't share an ancestor):

```ns-tree
Template
  Home
  About
    Team
    Mission
  Contact
Customer 42
  Home
  About
    Team
    Mission
  Contact
```

Every cloned row has a new primary key, regenerated `lft` / `rgt` / `depth` / `parent_id`, and the same business attributes (`name`, etc.) as its source. Aggregates roll back up automatically; observers and per-row Eloquent `created` events are suppressed for the cloned rows in favour of one [`SubtreeCloned`](../reference/events.html#subtree-cloning) event when the transaction commits.

## cloneSubtreeTo — clone under a parent

```php
$template = Category::query()->where('name', 'Template')->first();
$target   = Category::query()->where('name', 'Customer 42')->first();

$root = $template->cloneSubtreeTo($target);
// $root is the new root of the cloned subtree, already placed
// under $target as its last child.
```

The default is to append. Pass `$position` to control placement (same semantics as [`moveTo`](inserting.html)):

```php
$template->cloneSubtreeTo($target, position: 'first');
$template->cloneSubtreeTo($target, position: 2);   // 0-indexed
```

The optional `$transform` closure rewrites each row's raw attributes before insert. It receives the source row's attributes (no casts) plus the destination depth relative to the clone's new root (0 for the clone root, 1 for its direct children, …):

```php
$template->cloneSubtreeTo($target, transform: function (array $attributes, int $depth): array {
    $attributes['name'] = sprintf('[copy] %s', $attributes['name']);
    return $attributes;
});
```

Structural columns (`lft` / `rgt` / `depth` / `parent_id`), scope columns, the primary key, and materialised-path columns are owned by the package — returning any of them from `$transform` throws `LogicException`. Aggregate columns are silently stripped (the deferred recompute fills them in).

## cloneSubtreeAsRoot — clone as a new root

Same shape, but the clone lands at the top level of the source's scope:

```php
$template->cloneSubtreeAsRoot();
$template->cloneSubtreeAsRoot(position: 'first');
$template->cloneSubtreeAsRoot(transform: $rewrite);
```

For scoped models the clone stays in the source's scope. To clone across scopes, edit the scope column in `$transform` — but you must call `cloneSubtreeTo()` with an explicit parent in the destination scope; `cloneSubtreeAsRoot()` always inherits the source's scope.

## Static helper

```php
Category::cloneSubtree($template, $target);
Category::cloneSubtree($template, $target, position: 'first', transform: $rewrite);
```

Convention is to pass the optional args by name. The static form is equivalent to `$template->cloneSubtreeTo($target, ...)`.

## Soft-deleted rows

By default, trashed source rows are silently skipped: a trashed root throws upfront and trashed descendants are omitted from the clone. Pass `includeTrashed: true` to materialise them as live rows on the destination side (`deleted_at` is always `null` on clones):

```php
$template->cloneSubtreeTo($target, includeTrashed: true);
```

A trashed *destination* parent always throws — cloning into a deleted parent would orphan the new rows on restore / `forceDelete()`.

## What's preserved, what's regenerated

| Column kind                                     | Behaviour on the clone                                                                   |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Primary key                                     | Regenerated (autoincrement or UUID, per model config)                                    |
| Structural (`lft`, `rgt`, `depth`, `parent_id`) | Regenerated to match the destination position                                            |
| Scope columns                                   | Copied from the destination parent's scope (or the source's, for `cloneSubtreeAsRoot()`) |
| Materialised path                               | Recomputed under the destination parent's path                                           |
| `deleted_at`                                    | Always `null` (clones are live)                                                          |
| `created_at` / `updated_at`                     | Refreshed to the clone's transaction time                                                |
| Aggregate columns                               | Stripped on insert, then filled by the deferred recompute                                |
| Every other column                              | Copied verbatim (after `$transform` runs)                                                |

## Guards

| Condition                                                      | Throws                        |
| -------------------------------------------------------------- | ----------------------------- |
| Source has no bounds (unplaced)                                | `UnplacedNodeException`       |
| Destination parent has no bounds                               | `UnplacedNodeException`       |
| Source is trashed and `includeTrashed: false`                  | `InvalidArgumentException`    |
| Destination parent is trashed                                  | `InvalidArgumentException`    |
| Destination is in source's own subtree (including self)        | `InvalidCloneTargetException` |
| Source and destination scopes differ                           | `ScopeViolationException`     |
| `$transform` returns a structural / scope / primary-key column | `LogicException`              |

## Event

A single [`SubtreeCloned`](../reference/events.html#subtree-cloning) event fires after the outermost transaction commits, carrying the source root, the clone root, the row count, and the `includeTrashed` flag. See the events reference for the full payload.
