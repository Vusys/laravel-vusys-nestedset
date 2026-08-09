# Tree Diff

`TreeDiff` compares two snapshots of a tree and produces a structured, serialisable description of what changed — rows **added**, **removed**, **moved**, and **modified**. The diff is pure data: build it from any two snapshots, inspect or transport it, then `apply()` it to a live table when (and where) you choose.

Useful for:

- **Importing edited trees.** A CMS or front-end posts back the whole tree; diff it against the current state and apply only the delta instead of rebuilding.
- **Previewing changes.** `apply(..., dryRun: true)` reports the statements it *would* run without touching the database.
- **Undo / audit.** `invert()` swaps a diff to its undo direction; `toArray()` / `jsonSerialize()` give you an audit record.

Snapshots accept the same shapes `toJsonTree()` / `fromJsonTree()` use, so the exporter output round-trips straight into a diff.

## Building a diff

```php
use Vusys\NestedSet\Diff\TreeDiff;

$diff = TreeDiff::between($before, $after);
```

Both `$before` and `$after` accept any of:

- `iterable<Model&HasNestedSet>` — e.g. `Category::get()`
- flat rows: `iterable<array>` where each row carries `parent_id`
- nested rows: `list<array>` where each row carries `children`
- decoded JSON in either form

Rows are matched across the two sides by an **identity**, `id` by default. Pass a different column — or a closure — as the third argument when the primary key isn't stable across environments (e.g. a CMS that re-issues keys):

```php
$diff = TreeDiff::between($before, $after, on: 'slug');

$diff = TreeDiff::between($before, $after, on: fn (array $row) => $row['external_id']);
```

Structural columns (`lft`, `rgt`, `depth`, `parent_id`, `children`, `id`) never count as content changes. Pass `ignoreColumns:` to exclude more (timestamps, computed columns):

```php
$diff = TreeDiff::between($before, $after, ignoreColumns: ['updated_at']);
```

## Inspecting a diff

```php
$diff->summary();   // ['added' => 2, 'removed' => 1, 'moved' => 3, 'modified' => 0]
$diff->isEmpty();   // true when nothing changed
$diff->toArray();   // fully serialisable; jsonSerialize() returns the same
```

The four change lists are public, each a list of readonly value objects:

| Property          | Type             | Shape                                                |
| ----------------- | ---------------- | ---------------------------------------------------- |
| `$diff->added`    | `list<Added>`    | `key`, `parentKey`, `attributes`, `siblingPosition`  |
| `$diff->removed`  | `list<Removed>`  | `key`, `parentKey`, `attributes`, `siblingPosition`  |
| `$diff->moved`    | `list<Moved>`    | `key`, `fromParent`, `toParent`, `toSiblingPosition` |
| `$diff->modified` | `list<Modified>` | `key`, `before`, `after`                             |

`siblingPosition` / `toSiblingPosition` is the row's **0-indexed rank within its parent's children** in that snapshot — a pure sibling reorder surfaces as `Moved` rows even when no row changes parent.

## Applying a diff

```php
$result = $diff->apply(Category::class);
```

`apply()` runs the change set against `$modelClass`'s table inside one transaction, with aggregate maintenance deferred to a single trailing pass. The model class is supplied at apply-time so the same diff can be replayed across environments.

Phases run in a fixed order — **add → move → remove → modify** — chosen so the delta applies cleanly:

- **Adds first** so a later move can target a freshly-added parent.
- **Moves before removes** so a child the diff keeps is re-parented out of a removed subtree *before* the remove's cascade would delete it with the parent.

Added and moved rows are placed at their recorded `siblingPosition`, so `apply()` reproduces the `after` snapshot exactly — sibling order included.

### Dry run

```php
$result = $diff->apply(Category::class, dryRun: true);

$result->dryRun;            // true
$result->plannedStatements; // [['statement' => 'insert+gap', 'rows' => 1], ...]
```

No rows are touched; `plannedStatements` describes the work the real apply would do.

### Custom identity resolution

When identity isn't the primary key, `apply()` resolves each identity to a primary key with a single `whereIn` against the `on` column. Supply a `resolver` closure for composite keys or scoped lookups:

```php
$diff->apply(Category::class, resolver: fn (mixed $identity) => $lookup[$identity] ?? null);
```

## Inverting a diff

```php
// Capture the undo before applying, so you can roll the change back later.
$undo = $diff->invert();

$diff->apply(Category::class);
// … later …
$undo->apply(Category::class);
```

`invert()` swaps the diff to its undo direction — `added` ↔ `removed`, `Modified::before` ↔ `after`, `Moved::fromParent` ↔ `toParent`.

## Gotchas

- **Pure-reorder undo is lossy.** `Moved` records only the *destination* sibling position, not the origin, so `invert()` restores a reordered row's parent but not its original slot. Re-parenting undo is exact.
- **`invert()` only undoes what it was told.** A `Removed` constructed by hand without `attributes` / `parentKey` inverts to an `Added` with empty attributes — `between()` populates those fields for you.
- **Cross-environment identity.** Use `on:` (and a `resolver` at apply time) whenever primary keys differ between where the diff was built and where it's applied.
