# Changelog

All notable changes to `vusys/laravel-nestedset` are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html). Pre-1.0, backwards-compatibility breaks are allowed when called out under **Changed** / **Removed**.

## [Unreleased](https://github.com/vusys/laravel-nestedset/compare/v0.24.4...HEAD)

## [0.24.4](https://github.com/vusys/laravel-nestedset/compare/v0.24.3...v0.24.4) - 2026-07-10

Patch: one aggregate-correctness fix for a display-rounding drift that only surfaced on MySQL 9.

### Fixed

- **The delta-maintained `AVG` display column no longer drifts by one unit in the last decimal on MySQL 9.** The column was written as `1.0 * sum / NULLIF(count, 0)`; the scale-1 multiplier gave the quotient scale `1 + div_precision_increment`, which was then rounded a *second* time when stored into the `DECIMAL(_, 4)` column. The two roundings disagree at a 4th-decimal half-boundary — MySQL 8 rounded half-down (matching a fresh `AVG()` read) but MySQL 9 rounds half-up, so e.g. `1160/11` stored as `105.4546` while a fresh single-rounded `AVG()` returned `105.4545`, reported as aggregate drift. The coercion multiplier is now scale 6 so the quotient carries enough scale that the single final round into the column matches the fresh value on every backend. Weighted-avg is unchanged (its store and fresh sides already share the same shape). Surfaced by the non-gating MySQL 9 ceiling lane.

## [0.24.3](https://github.com/vusys/laravel-nestedset/compare/v0.24.2...v0.24.3) - 2026-07-08

Patch: three tree-correctness fixes, all surfaced by the new Runabout journey suite (seeded, shuffled, shrinking property tests for order-dependent invariants) — two `TreeDiff::apply()` corruptions on add/remove diffs, and a soft-delete `restore()` that could leave a live child under a trashed parent.

### Changed

- **`restore()` now rejects a node whose parent is still soft-deleted**, throwing the new `TrashedAncestorException`. The restore cascade only walks *down* (it brings back the anchor plus its same-stamp descendants but never restores ancestors), so restoring a mid-subtree node left a live child parented under a trashed one — the exact "live child under a trashed parent" state the insert / factory path already refuses with `TrashedTargetException`. The guard keeps that invariant total in both directions: restore **outward-in** (parent before child). Restoring from the top of a trashed subtree (parent live, or a root) is unchanged, and a trashed child under a *live* parent is still fine. The check is a single indexed lookup on the parent's `deleted_at` and runs before any write, so a rejected restore leaves the tree and its aggregates untouched. Closes #218; surfaced by the soft-delete Runabout journey.

### Fixed

- **The former force-delete-after-partial-restore aggregate drift is now unreachable.** Force-deleting a trashed parent that had an individually-restored live child used to leave the ancestor chain counting a destroyed row. Its precondition — a live child under a trashed parent — can no longer be constructed now that the mid-sequence `restore()` throws, so the downstream drift can't occur.
- **`TreeDiff::apply()` corrupted the tree when a diff's removed set contained both a node and one of its descendants, on a hard-delete model.** `TreeDiffApplier::doRemoves()` loaded every removed row up front, then deleted each using stale in-memory bounds. Deleting an ancestor cascade-removes its descendants and closes their gaps, so the loop then re-deleted an already-gone descendant with stale bounds, double-closing an ancestor's `rgt`. Each removed row is now reloaded against the live tree immediately before its delete: rows a prior cascade already took are skipped, and survivors close their gap with current bounds. Soft-delete models were unaffected.
- **`TreeDiff::apply()` threw `moveToSiblingPosition(): position must be in [0, N]` when a diff added a node at a sibling position that a later move would fill.** `apply()` runs its phases add → move → remove → modify, so when a parent's final children come from a mix of added and moved-in nodes, an added node's recorded position could exceed the child count present during the adds phase. The applier now clamps the add-time position to the current tail and lets the moves phase settle the final sibling order.

## [0.24.2](https://github.com/vusys/laravel-nestedset/compare/v0.24.1...v0.24.2) - 2026-07-07

Patch: stop a bounds-stale instance from drifting aggregates on a source-column update.

### Fixed

- **Updating an aggregate source column on a held instance whose `lft`/`rgt` had gone stale banded the delta onto the wrong ancestor chain.** A source-column save rewrites only the dirty column, never the structural bounds, so an instance shifted by an earlier move/append (but not refreshed) still carried its old bounds while the DB row held the current ones. The `saved`-hook delta pass keyed its ancestor `UPDATE` off those stale bounds — missing the node itself and spilling the delta onto whatever now occupied the old position, producing permanent silent drift. The delta pass now re-reads `lft`/`rgt`/`depth` from the database before banding (locking the row when a transaction is open), matching the discipline already used on the delete, move and restore paths. Found by the scheduled aggregate stale-instance fuzzer.

## [0.24.1](https://github.com/vusys/laravel-nestedset/compare/v0.24.0...v0.24.1) - 2026-07-05

Patch: fail loud on a misconfigured `NodeTrait` model instead of silently corrupting the tree.

### Added

- `MisconfiguredNodeException`, plus a `@phpstan-require-implements MaintainsTreeAggregates` constraint on `NodeTrait`. A model that composes the trait but omits `implements MaintainsTreeAggregates` is now caught both statically (`composer analyse`) and at runtime on the first `save()`.

### Fixed

- **A `NodeTrait` model missing `implements MaintainsTreeAggregates` used to insert rows with `lft = rgt = 0` and no error.** Every lifecycle listener gated on that interface, so `saveAsRoot()` / `appendToNode()->save()` placed nothing while appearing to succeed (`bulkInsertTree()` masked it by writing the bounds attributes directly). The `saving` listener now throws `MisconfiguredNodeException` instead of silently producing an `invalid_bounds` row.

## [0.24.0](https://github.com/vusys/laravel-nestedset/compare/v0.23.0...v0.24.0) - 2026-06-13

Audit follow-ups on v0.23.0: cross-backend correctness (reserved-word columns, MariaDB fresh-reads, MySQL/MariaDB index-name limits), soft-delete structural semantics, concurrency hardening, and new coherence fuzzers. Validated across SQLite, MySQL, MariaDB and PostgreSQL.

### Added

- Secondary `[scope…, parent_id]` index in the `nestedSet()` Blueprint macro (decision #6). The shipped composite index leads with `lft`/`rgt` and can't serve a `parent_id` lookup — `children()`, `whereIsRoot()` and `fixTree()`'s parent walk. MySQL gets one free off its FK; PostgreSQL and SQLite did not. Dropped by `dropNestedSet()`.
- `TrashedTargetException` — placing a live node relative to a soft-deleted anchor now throws instead of stranding a live-descendant-of-trashed that `restore()` can never reconcile.
- Materialised-path coherence fuzzer and a stale-instance pool in the aggregate fuzzer, both asserting incremental maintenance matches a full recompute after random mutation sequences.

### Changed

- **`appendToNode` / `prependToNode` / `insertBeforeNode` / `insertAfterNode` onto a soft-deleted target now throw `TrashedTargetException`** (was silently allowed).
- **`reorderChildrenBy()` now reasons over the raw sibling set** (live + trashed), matching `reorderChildren()` — previously it built its id list from a live-only query, which `reorderChildren()` then rejected as "missing children".
- `nestedSet()` index names are now bounded to 64 characters (MySQL/MariaDB identifier cap) via `boundedIndexName()`: existing schemas keep Laravel's native name verbatim; longer table+scope combinations get a deterministic hashed name. Previously a long composite overran the limit, the `ALTER … ADD INDEX` failed (error 1059), and the half-built table survived (non-transactional DDL), breaking every subsequent migration.
- `TreeDiff` orders flat-snapshot siblings by `lft` (not input arrival order), so two snapshots of the same state read with different `orderBy` no longer diff as a sea of phantom `Moved` entries. Falls back to input order when `lft` is absent.

### Fixed

- Structural columns (`lft`/`rgt`/`depth`/`parent_id`) are now grammar-quoted in the mutation engine, repair path, and the `even_bounds_width` corruption check. A model renaming a structural column to a SQL reserved word (`left`/`order`) no longer emits a syntax error on every mutation, `fixTree()` or `countErrors()` on PostgreSQL/MySQL/MariaDB.
- MariaDB: `withFreshAggregates()` combined with `->limit()` / `->offset()` (pagination, top-N) no longer hard-errors (1235) — `LIMIT`/`OFFSET`/`ORDER` are stripped from the id-membership subquery (the outer query still bounds the result).
- Soft-delete cascade now works under the `immutable_datetime` cast — the timestamp matcher broadened from Carbon to `DateTimeInterface`, so the cascade no longer silently no-ops and leaves descendants live.
- Subtree materialised-path rewrite: scope predicates are now bound in forward column order (a multi-column scope previously cross-bound its columns and matched no rows), and the `SUBSTRING` offset uses `mb_strlen()` so multibyte path prefixes aren't cut mid-character.
- The soft-delete `deleting` hook re-reads structural bounds under `FOR UPDATE`, matching every other stale-bounds path — closes a window where a concurrent gap-shift could make the delete close the wrong gap (skipped on SQLite / when no transaction is open).

### Documentation

- Documented that `aggregate_locking` `'auto'` is not fully race-free on PostgreSQL `READ COMMITTED` (the recompute's correlated subqueries read the statement snapshot), and that crossing moves lock in operation order and so can deadlock — with the standard retry-on-deadlock remedy.
- Pinned (skipped) and documented the force-delete-after-restore aggregate drift limitation, with the `fixAggregates` workaround.

## [0.23.0](https://github.com/vusys/laravel-nestedset/compare/v0.22.0...v0.23.0) - 2026-06-11

A correctness, concurrency and pre-1.0 hardening pass. Several backwards-compatibility breaks are called out under **Changed** / **Removed** — pre-1.0, these are allowed. Validated across SQLite and MySQL (full suite + every seeded fuzzer).

### Added

- `NestedSetException` marker interface, implemented by every exception the package throws — `catch (NestedSetException $e)` now catches any package-originated failure regardless of its SPL base class.
- `countErrors()` detects two more invariant violations: `overlapping_bounds` (sibling intervals that partially overlap without nesting) and `even_bounds_width` (a structurally impossible even-width span).
- Scoped **root seeding**: `bulkInsertTree($tree, null, $scope)` and `fromJsonTree()` (which reads the scope from the root rows) can now seed new trees on a `#[NestedSetScope]` model without an anchor.
- `includeKeys` is now functional in `bulkInsertTree()` / `fromJsonTree()` — rows may carry their own primary keys; a duplicate surfaces as `JsonImportKeyCollisionException` (previously unreachable).
- PostgreSQL advisory lock serialising concurrent first-root `makeRoot()` calls within an empty scope (MySQL/MariaDB already serialise via next-key gap locks; SQLite is single-writer).
- Fork-based concurrency tests for moves, reorders and empty-scope first-root creation.

### Changed

- **`moveToSiblingPosition()` is now 0-based** (was 1-based), matching `moveTo()`, `TreeDiff` sibling positions and the factory.
- **`toTree()` / `toFlatTree()` return a forest** on a partial/filtered fetch: a node whose parent isn't in the collection becomes a top-level node instead of being silently dropped. An explicit `$root` still narrows to that subtree.
- **Chaining two placement calls before `save()` now throws** (e.g. `appendToNode($a)->insertAfterNode($b)`) instead of silently dropping the first.
- **Changing a scope column on an existing node now throws** `ScopeViolationException` on save (the scoped mutation would otherwise shift the wrong partition).
- **`children()` no longer applies a scope predicate** — it keys purely on `parent_id` (a globally-unique reference), which fixes eager-load / `withCount` / `whereHas` on scoped models.
- **Inspection predicates** (`isLeaf`, `isDescendantOf`, `isAncestorOf`) return `false` on a never-placed node instead of throwing.
- **`freshAggregate()` casts its result** through the model's column cast, so it is type/precision-stable across drivers and directly comparable to the stored attribute.
- **Cloning a node whose materialised path would collide** now throws `DuplicatePathSegmentException` (was a silent duplicate); the path rebuild runs inside the clone transaction.
- The mutation cycle-guard and the `*Scope` exporters now throw `CyclicMoveException` / `ScopeViolationException` (both extend `LogicException`) instead of bare `LogicException`.

### Removed

- **Renamed the path exceptions** to the `Exception` suffix used by every other exception: `DuplicatePathSegment`, `EmptyPathSegment`, `InvalidPathSegment`, `NonDeterministicPathSegment`, `PathTooLong` → `*Exception`.

### Fixed

- **Per-model column-name overrides** (`getLftName()` etc.) are now honoured across the whole read layer — `TreeQueryBuilder` delegated to global config only, breaking renamed-column models.
- **Stale in-memory aggregate state** caused silent drift: moving or deleting a held instance transferred/subtracted stale totals, and the restore cascade banded on stale bounds. Move, delete and restore now re-read fresh values/bounds.
- **`withDeferredAggregateMaintenance()` with a mid-tree anchor** left ancestors above the anchor drifted — the closing repair now runs from the tree root.
- **Anchored `fixTree()`** rebuilt materialised paths against the anchor's stale pre-repair bounds; it now re-reads them.
- **Filter-predicate cast asymmetry** (a boolean cast over a TINYINT filter column) and the listener old-snapshot path drifted; both sides now use cast-consistent / raw-original values.
- **Concurrent moves** read the mover's own bounds without a lock — now `FOR UPDATE`. `reorderChildren()` runs its reads inside the locked transaction, and a move into the node's own descendant is rejected before the before-move aggregate hook.
- **`JsonAgg` / `JsonObjectAgg` drift detection** is now order-insensitive (MySQL's `JSON_ARRAYAGG` can't honour `ORDER BY`).

## [0.22.0](https://github.com/vusys/laravel-nestedset/compare/v0.21.0...v0.22.0) - 2026-06-11

### Fixed

- **Soft-delete cascade marker format diverged from the anchor row** (anchor `…12:00:00` vs descendants `…12:00:00.000000`), so restoring an outer ancestor could restore part of an interleaved cascade and strand a trashed node under a live parent (diverging across backends). The cascade now stamps descendants with the anchor's exact seconds-precision value.
- **Anchored `fixTree()` hung / OOM'd on `parent_id` cycles** — the documented cycle-recovery tool now terminates (visited-set guards in the subtree walk).
- **MIN/MAX aggregates drifted** on `NULL ↔ value` source updates and listener-contribution transitions (the SUM-style `NULL → 0` coercion leaked into the MIN/MAX update paths).
- **Stale in-memory values on the move path**: `insertAfterNode`/ `insertBeforeNode` could stamp a stale `parent_id`, and the before-move aggregate hook used stale bounds. Both now read fresh.
- **A `saving`/`creating` listener returning `false` committed structural SQL** (gap/move) without the row write; the cancel now rolls back.
- **`delete()`/`forceDelete()` were not wrapped in the auto-transaction**, so a throw mid-pipeline left a permanent hole — now wrapped like `save()`.
- **Hard-deleting an unplaced row shifted every placed row in scope**; `saveQuietly()` silently dropped queued placements — both now guarded.
- **`bulkInsertTree()` trusted the anchor's stale in-memory bounds** — now reads + locks the anchor row inside its transaction.
- **`withDeferredAggregateMaintenance()` swallowed repair failures on the success path** — the failure now propagates.
- **`TreeDiff::apply()`** discarded recorded sibling positions and deleted retained children before moving them; **`fromJsonTree()`** returned the wrong nodes (DFS pre-order slice bug).
- **`isDescendantOf()`/`isAncestorOf()`** ignored scope (cross-tree false positives); **relations** had no `ORDER BY`; **`#[NestedSetScope]`** was not inherited by subclasses; `withDepth()`/`whereIsLeaf()` didn't wrap identifiers; JSON-import collision keys dropped UUID/string PKs.

### Added

- `countErrors()` detects three more corruption categories: `parent_bounds_mismatch`, `depth_mismatch`, `bounds_out_of_range`.
- `fixTree($anchor)` rejects an unplaced anchor and reports the rows it actually walked in `TreeFixResult::nodesUpdated`.
- Documentation for the Tree-diff subsystem.

### Removed

- Dead bitwise-delta maintenance paths (all bitwise aggregates already maintain via chain recompute); the docs now reflect this.

## [0.21.0](https://github.com/vusys/laravel-nestedset/compare/v0.20.0...v0.21.0) - 2026-06-04

### Added

- Top-K aggregate (`topK`).
- Lazy aggregates with TTL.
- Aggregate change-feed event (`NestedSetAggregateChanged`).
- Listener-aggregate hardening: O(N) repair, `filter:`, variance / stddev / geometric-mean / harmonic-mean.

### Fixed

- Type-preserving MIN/MAX reads; guard unplaced mutation targets; cross-backend mutation + scope-resolver gaps.

## [0.20.0](https://github.com/vusys/laravel-nestedset/compare/v0.19.0...v0.20.0) - 2026-05-31

### Added

- Materialised-path columns.

## [0.19.0](https://github.com/vusys/laravel-nestedset/compare/v0.18.2...v0.19.0) - 2026-05-31

### Added

- Tree diff + JSON tree import.
- Subtree cloning (`cloneSubtreeTo`, `cloneSubtreeAsRoot`).
- Sibling reorder primitive (one `CASE WHEN` UPDATE per reorder).
