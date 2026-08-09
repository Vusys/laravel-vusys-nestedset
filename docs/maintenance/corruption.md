# Tree Corruption — Detection, Recovery, Prevention

This document explains how a nested-set table managed by this package can become inconsistent, what the package can repair automatically, what it cannot, and how to avoid getting into the bad state in the first place.

If you only have time for one paragraph: **`parent_id` is treated as authoritative.** Every repair the package does works by walking the tree implied by `parent_id` and rebuilding `lft` / `rgt` / `depth` from that walk. So as long as `parent_id` describes the tree you actually want, every other column is recoverable. If `parent_id` itself is wrong (cycles, lost references, cross-scope pointers), no automated repair can guess the intended shape.

## Background: the invariants

A well-formed nested-set tree (in a single scope) satisfies all of:

| Invariant                          | Statement                                                                           |
| ---------------------------------- | ----------------------------------------------------------------------------------- |
| **Strict bounds**                  | `lft < rgt` for every row.                                                          |
| **Unique lft / rgt**               | `lft` values are unique across the scope; `rgt` values too.                         |
| **Contiguous numbering**           | `{lft, rgt}` values form a permutation of `1 .. 2N`.                                |
| **Containment ⇔ ancestry**         | Row B is a descendant of A *iff* `A.lft < B.lft < B.rgt < A.rgt`.                   |
| **`parent_id` agrees with bounds** | If B is a child of A, then B's `parent_id = A.id`.                                  |
| **No cycles**                      | Following `parent_id` from any row eventually reaches `null` (a root).              |
| **`parent_id` stays in scope**     | For scoped models, a row's `parent_id` references a row with the same scope values. |
| **`depth` matches `parent_id`**    | A node's `depth` is one more than its parent's (or `0` for a root).                 |

When the package's own API is the only thing that mutates the table, all of these are maintained automatically. Corruption means at least one of them has been broken — typically by a write that bypassed the package.

## Detection

Two methods detect violations:

```php
// Structural invariants (returns counts; key names listed below).
$errors = Category::countErrors();
// → ['invalid_bounds' => 0, 'duplicate_lft' => 0, 'duplicate_rgt' => 0, 'orphans' => 0,
//    'parent_bounds_mismatch' => 0, 'depth_mismatch' => 0, 'bounds_out_of_range' => 0]

Category::isBroken(); // true if any of the above is > 0

// Aggregate-column drift (returns counts per aggregate column).
$drift = Category::aggregateErrors();
// → ['articles_total' => 3, 'articles_max' => 0, ...]

Category::aggregatesAreBroken(); // true if any column has drift
```

On scoped models pass an anchor: `Category::countErrors($root)`. Without one the call throws `ScopeViolationException` to stop you from accidentally counting errors across every tenant's tree at once.

The structural check is one indexed query per category — range scans, a `GROUP BY` for the duplicate counts, and a handful of self-joins (`parent_bounds_mismatch`, `depth_mismatch`, cross-column collisions). The aggregate-drift check is the same cost as one `withFreshAggregates` pass.

> **Cycles are not currently surfaced by `countErrors()`.** They appear indirectly as "rows you couldn't see in the tree after a repair" — see [`parent_id` cycles](#parentid-cycles). Detection SQL is in [Diagnostic SQL](#diagnostic-sql).

> **Visualise the damage.** When `countErrors()` returns non-zero, `dd($root->toAsciiTree())` or `Category::toMermaidForest()` often makes the damage obvious at a glance. The exporters fold by `parent_id` and throw `CorruptTreeException` on cycles, so the output matches what `fixTree()` would rebuild. See [Tree Exporters](../querying/exporters.md).

## Corruption categories

### `invalid_bounds`

**Symptom.** A row has `lft >= rgt`.

**Meaning.** The row claims to occupy no space (or negative space) in the tree's interval. Any descendant predicate involving it returns nonsense. Reads do not crash, but `whereDescendantOf` for that node yields zero or arbitrary rows.

**Typical causes.**

- A direct `DB::table('categories')->update(['lft' => …, 'rgt' => …])` that swapped or mis-ordered the values.
- A failed mutation in a non-transactional database engine (MySQL with `MyISAM`, or any backend with a crashed `BEGIN ... COMMIT`).
- A migration that copied an old kalnoy/nestedset table over without rebuilding bounds.

**Repair.** Automatic. `fixTree()` rebuilds `lft`/`rgt`/`depth` from `parent_id` so the affected row gets a fresh, consistent interval.

### `duplicate_lft` / `duplicate_rgt`

**Symptom.** Two rows in the same scope share the same `lft` value (or the same `rgt` value).

**Meaning.** The interval encoding is broken — those two rows claim to start (or end) at the same numeric position. `whereAncestorOf` and `whereDescendantOf` predicates return both rows when they should return one. Subtree aggregates double-count.

**Typical causes.**

- Two parallel writes opening overlapping gaps without locking — the package wraps each mutation in `config('nestedset.auto_transaction')` by default; turning that off and not adding your own locking can produce this.
- A partial `makeGap` that ran on some rows but not others (crashed mid-statement on a non-transactional engine).
- Manually inserted rows with `lft`/`rgt` literals that collided with existing ones.

**Repair.** Automatic. `fixTree()` rebuilds the numbering.

### `orphans`

**Symptom.** A row's `parent_id` is non-null but no row with that id exists in the same scope.

**Meaning.** The row claims a parent that isn't reachable. Reads treat the row as an unaffiliated stub. **`fixTree()` does not "fix" this** — see [Not-automatically-recoverable cases](#not-automatically-recoverable-cases).

**Typical causes.**

- Direct `DELETE` statements that bypass Eloquent and the trait (the package's `delete()` and `forceDelete()` paths both cascade to descendants — raw SQL does not).
- A scoped move that didn't update the child's scope column — the child still references a parent id that exists, but in a different scope, which the orphan query rightly counts as missing.

**Repair.** Partial. `fixTree()` ignores orphans during the walk, so their `lft`/`rgt`/`depth` remain whatever they were before. To actually remove the orphan condition you must either:

1. **Re-parent** them via `appendToNode()` / `prependToNode()` to an existing parent, then run `fixTree()`.
2. **Promote to root** via `makeRoot()->save()`, then `fixTree()`.
3. **Delete** them via `Model::find($orphanId)->delete()`.

The package can't pick the right answer because the right answer is domain-specific.

### `parent_bounds_mismatch`

**Symptom.** A child's bounds are not strictly inside its (existing) parent's bounds — i.e. `lft`/`rgt` disagree with the `parent_id` they claim.

**Meaning.** This is the corruption a direct `parent_id` edit produces: the row points at a new parent but its interval still sits where the old one was (or a raw `lft`/`rgt` edit moved the interval out from under a valid `parent_id`). The bounds are internally consistent — `lft < rgt`, no duplicates — so the bounds-only checks read clean, but the tree the bounds describe contradicts the tree `parent_id` describes.

**Typical causes.**

- A raw `UPDATE … SET parent_id = …` re-parenting a row without re-running `fixTree()`.
- A partial mutation that wrote `parent_id` but crashed before the gap shift.

**Repair.** Automatic. `fixTree()` rebuilds `lft`/`rgt`/`depth` from `parent_id`, which is authoritative.

### `depth_mismatch`

**Symptom.** A row's stored `depth` isn't `parent.depth + 1` (or isn't `0` for a root).

**Meaning.** `depth` is stored and maintained, not recomputed on read, so a raw edit or a half-applied move can leave it drifting from the structure. Queries that use `depth` (e.g. `withDepth` comparisons, level filters) then return wrong levels.

**Typical causes.**

- A raw `lft`/`rgt`/`parent_id` edit that didn't also fix `depth`.
- Importing a tree that carried its own `depth` values inconsistent with the package's convention.

**Repair.** Automatic. `fixTree()` reassigns `depth` from the walk.

### `bounds_out_of_range`

**Symptom.** A placed row has a bound below `1` (a stray `0`), or a value that is simultaneously one row's `lft` and another's `rgt` — a cross-column collision, e.g. `X(0, 1)` sitting alongside `Root(1, 4)` (the value `1` is both `X.rgt` and `Root.lft`).

**Meaning.** In a valid nested set every `lft`/`rgt` value is distinct and `>= 1`, so the `lft` and `rgt` value sets are disjoint. A below-`1` bound or a cross-column collision breaks that in a way `lft >= rgt` and the per-column duplicate checks miss. The check is **gap-tolerant**: a sparse but otherwise valid tree (reserved slots not yet filled) is not flagged, and fully-unplaced rows (`lft = rgt = 0`) are excluded — they only arise from raw SQL inserts that bypass the model (saving an unplaced node through the normal API throws `UnplacedNodeException`), so they aren't treated as corruption.

**Typical causes.**

- A raw `lft`/`rgt` literal that landed below `1` or overlapped an existing bound.
- A partially-applied gap shift that left two rows sharing a boundary value.

**Repair.** Automatic. `fixTree()` renumbers the whole scope into a fresh dense permutation.

### `parent_id` cycles

**Symptom.** Two or more rows form a cycle through `parent_id` (e.g. `A.parent_id = B.id`, `B.parent_id = A.id`). None of them is `parent_id = null`, but none reaches `null` by walking parents either.

**Meaning.** The "tree" implied by `parent_id` is no longer a tree. There is no consistent `(lft, rgt, depth)` assignment that satisfies the invariants. `fixTree()` walks **from roots only** — rows in a cycle are not roots and are not reachable from any root, so they are silently skipped by the rebuild and keep their stale bounds.

**Typical causes.**

- A `update parent_id =` statement that swapped two rows' parents, introducing a 2-cycle. The package's own API rejects this: the `insertAfterNode` / `appendToNode` family validates that you're not moving a node into its own subtree. Bypassing those guards is the only way to create a cycle.
- Imported data from another source where the parent column wasn't validated for acyclicity.

**Repair.** Not automatic. There is no way for the package to know which edge in the cycle is "wrong". Diagnostic SQL is in [Diagnostic SQL](#diagnostic-sql). Common manual fixes:

1. Pick one row in the cycle and `update categories set parent_id = null` on it (promote it to root). The cycle is now broken; the others become children of the new root. Then run `fixTree()`.
2. Identify the row whose `parent_id` change introduced the cycle (via audit logs or git blame on the data-fix script) and reset that one row's `parent_id` to its correct historical value.

### Aggregate drift

**Symptom.** `Category::aggregateErrors()` reports a non-zero count for one or more aggregate columns.

**Meaning.** A stored aggregate column (`articles_total`, `articles_min`, …) disagrees with what it would be if recomputed from the source column right now.

**Typical causes.**

- Direct `DB::table('categories')->update(['articles' => …])` that bypassed the trait's `saved` / `created` / `deleted` hooks.
- Raw `INSERT … VALUES (…)` of new nodes that didn't go through Eloquent.
- A migration that altered the source column without rebuilding aggregates.

**Repair.** Automatic. `Category::fixAggregates()` overwrites stored values with freshly-computed ones from the source. The structural tree must already be intact — drift is computed by joining each row to its subtree, so corrupt bounds give garbage results. Run `fixTree()` first if both have happened.

### Materialised-path drift

**Symptom.** A declared materialised-path column (`url_path`, `crumb_path`, etc.) holds a value that doesn't match the path derived by walking `parent_id` to the root.

**Meaning.** The stored path has diverged from the structural tree. The structural columns themselves may still be intact.

**Typical causes.**

- Direct `DB::table('categories')->update(['url_path' => …])` that bypassed the saving listener.
- A bulk job wrapped in `withoutMaterialisedPathMaintenance()` that didn't run a follow-up repair.
- Backfill rows imported from a pre-feature dataset.
- An edit to the source attribute (`name`, `display_name`, …) inside a transaction that rolled back, leaving descendants' paths inconsistent.

**Repair.** Automatic. `Category::fixMaterialisedPaths()` walks parent\_id and rebuilds every declared column for every row. Pass a column name to limit the scope. The structural tree must already be intact — the rebuild reads `parent_id` and trusts it. Run `fixTree()` first if both have happened (`fixTree()` calls the path rebuild as its final step, so a single `fixTree()` covers both kinds of drift). See [Materialised Paths](../tree-operations/materialised-paths.md) for the wider feature reference.

## Recovery

### What `fixTree()` actually does

```php
Category::fixTree();                // forest (unscoped)
Category::fixTree($rootCategory);   // single tree, scoped or anchored
```

1. Reads every row in scope into memory (just `id` and `parent_id`).
2. Walks the tree top-down from rows where `parent_id IS NULL`, numbering `lft` / `rgt` in pre-order and assigning `depth` from the walk's recursion level.
3. Issues per-row `UPDATE`s inside one transaction to write the new bounds.
4. Re-checks `countErrors()` and returns a `TreeFixResult` so you can see what's still broken.

Two important consequences:

- **`parent_id` is read, not written.** Whatever `parent_id` you have is what the rebuild trusts. Bad `parent_id` ⇒ bad rebuild.
- **Unreachable rows aren't touched.** Orphans ([orphans](#orphans)) and rows in cycles ([`parent_id` cycles](#parentid-cycles)) are silently left with their pre-repair bounds. The `TreeFixResult.errors` count after the run will still show them.

### What `fixAggregates()` actually does

```php
Category::fixAggregates();          // forest
Category::fixAggregates($root);     // single subtree
```

1. For every row in scope, computes the freshly-aggregated value of each declared aggregate column (`SUM`/`COUNT`/`AVG`/`MIN`/`MAX` over the subtree, inclusive by default).
2. Compares the freshly-computed value to the stored value.
3. For every row whose stored value disagrees, issues a single chunked `UPDATE … SET col = CASE id WHEN … END WHERE id IN (…)` to write the correction.
4. Returns an `AggregateFixResult` with per-column counts of rows updated.

For large drifted trees where the synchronous wait would block a web request, dispatch the same work to a queue:

```php
Category::queueFixAggregates();           // unscoped
Category::queueFixAggregates($anchor);    // scoped
```

For an offline command that wants to stream progress instead, pass `chunkSize` (and optionally `onChunk`) to the synchronous method:

```php
Category::fixAggregates(
    chunkSize: 1_000,
    onChunk: fn ($r, $i, $cur) => $this->output->writeln("chunk {$i}: {$r->totalRowsUpdated}"),
);
```

Both paths share the same underlying chunking machinery; see [Repairing Aggregates](fix-aggregates.html) for routing options on the queued form, chunked self-redispatch, and deferred maintenance.

Idempotent: running it twice in a row, the second invocation finds zero drift and updates nothing. Safe to call after every batch operation as belt-and-braces.

### Recovery cheat-sheet

| What you observe                                   | Run this                                               | Then                                                                                                                        |
| -------------------------------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `isBroken() === true`                              | `fixTree()`                                            | Re-run `countErrors()` — orphans/cycles will still show.                                                                    |
| `aggregatesAreBroken() === true`, structure intact | `fixAggregates()`                                      | Done.                                                                                                                       |
| Both broken                                        | `fixTree()`, then `fixAggregates()`                    | `fixTree()` calls `fixAggregates()` for you internally — but only on the post-rebuild structure, so the order is important. |
| Materialised-path columns disagree with the tree   | `fixMaterialisedPaths()`                               | Pass a column name to limit; `fixTree()` already runs this as its final step.                                               |
| Orphans after `fixTree()`                          | Re-parent, root-ify, or delete per [orphans](#orphans) | Then `fixTree()`.                                                                                                           |
| Cycles after `fixTree()`                           | See [`parent_id` cycles](#parentid-cycles)             | Diagnostic SQL in [Diagnostic SQL](#diagnostic-sql).                                                                        |

## Not-automatically-recoverable cases

The package cannot guess the right answer for:

1. **`parent_id` cycles.** No way to pick which edge in the cycle is the bogus one.
2. **Orphans** (in the sense of automatically *clearing* the orphan condition). Detected, but cleaning up requires a domain decision.
3. **Lost source values.** If the column an aggregate is computed over (e.g. `articles`) has itself been corrupted, `fixAggregates` will dutifully recompute the wrong answer. Aggregates can only be as accurate as their source.
4. **Cross-scope `parent_id`.** A scoped model whose row points its `parent_id` at a row in a different scope is treated as orphan by the same-scope orphan check — and recovery options are the same as [orphans](#orphans).
5. **Schema drift.** If `lft`/`rgt`/`parent_id`/`depth` columns were renamed in the database but not in the model (or vice versa), the package will silently operate on stale state. Always keep the migration and `config('nestedset.columns')` in sync.

## Prevention

In order of impact:

1. **Mutate only through Eloquent on a `NodeTrait` model.** Every `appendToNode` / `prependToNode` / `insertBeforeNode` / `insertAfterNode` / `makeRoot` / `delete` / `forceDelete` / `restore` path is wrapped in a transaction and maintains every invariant. The corruption taxonomy above is almost entirely reachable only by bypassing this surface.
2. **For bulk loads, do one of:**
   - **Wrap in `Model::withDeferredAggregateMaintenance(Closure)`** when the loop goes through Eloquent (so events, mutators, casts still fire). The per-row aggregate maintenance defers, one `fixAggregates` runs at the end — no per-save ancestor UPDATEs.
   - Let Eloquent handle every row without the wrapper — always correct, but every save touches the ancestor chain.
   - Build the table with raw INSERTs and run `fixTree()` once at the end — `parent_id` is the only column you need to get right; everything else will be rebuilt.
   - Use `Model::bulkInsertTree($rows, appendTo: $anchor)` which does the above as one operation — see [Bulk Insertion](../tree-operations/bulk-insertion.html).
3. **Don't write to `lft` / `rgt` / `depth` directly.** Treat them as derived. The only authoritative column is `parent_id` — editing that is fine (provided you keep it acyclic and in-scope) because `fixTree()` is always your recovery option.
4. **Keep auto-transactions on.** `config('nestedset.auto_transaction')` defaults to `true` and wraps the multi-step CASE WHEN UPDATEs in a transaction. Turning it off on a non-Postgres engine without supplying your own locking is the most reliable way to produce duplicate `lft`/`rgt` values under concurrency.
5. **Validate imports.** When loading from CSV / JSON / another ORM, run `countErrors()` and `aggregateErrors()` immediately after the load and fail fast if either is non-zero.
6. **`forceDelete` cascades through the trait — `DB::table(...) ->delete()` does not.** Always delete through the model when you care about descendants.
7. **For scoped models, never move a row between scopes via raw SQL.** The trait throws `ScopeViolationException` for cross-scope moves through the public API; raw SQL bypasses that guard.

## Diagnostic SQL

Useful one-liners when you're staring at a broken table.

**Find rows in a cycle (PostgreSQL / MySQL 8.0+ / MariaDB 10.2+).** Recursive CTE walking from null-parent roots; any row not visited is in a cycle, an orphan, or both.

```sql
WITH RECURSIVE walk AS (
  SELECT id, parent_id, 0 AS depth FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT a.id, a.parent_id, w.depth + 1
    FROM categories a INNER JOIN walk w ON w.id = a.parent_id
)
SELECT a.id, a.parent_id, a.name
  FROM categories a
  LEFT JOIN walk w ON w.id = a.id
 WHERE w.id IS NULL;
```

**Find orphans only** (parent\_id non-null pointing at a missing id):

```sql
SELECT child.id, child.parent_id, child.name
  FROM categories AS child
  LEFT JOIN categories AS parent ON parent.id = child.parent_id
 WHERE child.parent_id IS NOT NULL AND parent.id IS NULL;
```

> **Scoped tables.** For `#[NestedSetScope]` models, equate the scope columns inside the JOIN — `ON parent.id = child.parent_id AND parent.menu_id = child.menu_id` — and add `AND child.menu_id = ?` to the outer `WHERE`. A child whose `parent_id` happens to match a row in a *different* scope would otherwise join successfully and mask the orphan; the package's `orphanQuery()` (`src/Query/TreeRepairBuilder.php`) does this scope-equality on the JOIN for the same reason.

**Find aggregate drift on one column** (e.g. `articles_total`):

```sql
SELECT outer_a.id, outer_a.articles_total AS stored, agg.computed
  FROM categories AS outer_a
  LEFT JOIN (
    SELECT o.id AS outer_id, COALESCE(SUM(d.articles), 0) AS computed
      FROM categories AS o
      INNER JOIN categories AS d ON d.lft BETWEEN o.lft AND o.rgt
     GROUP BY o.id
  ) AS agg ON agg.outer_id = outer_a.id
 WHERE outer_a.articles_total <> agg.computed;
```

> **Soft deletes and scopes.** This is the canonical form for an inclusive `SUM(articles)` over an unscoped, hard-delete-only table. For tables with `SoftDeletes`, add `AND d.deleted_at IS NULL` to the inner join (and the same on `outer_a` if you want trashed ancestors to be skipped too) — the package's maintenance excludes soft-deleted descendants, so without the filter this query will report false drift. For scoped models, add the scope equality to both `o`/`d` and `outer_a`/`agg`. The package's `withFreshAggregates()` (`src/Query/Aggregates/Read/FreshAggregateProjector.php`) emits both predicates automatically — call it instead of writing the SQL by hand whenever you can.

**Spot-check a single node**: list its subtree via the lft predicate and via `parent_id` recursion, then compare row counts. Disagreement means either bounds are wrong (run `fixTree()`) or there's a cycle in that subtree.

---

*See also: [Tree Repair](fix-tree.html) for the public API surface and `tests/Feature/Corruption/` for executable examples of every category in §3.*
