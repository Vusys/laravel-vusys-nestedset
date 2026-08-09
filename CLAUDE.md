# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this package is

`vusys/laravel-nestedset` — a modern reimplementation (not a fork) of the nested-set model for Laravel. Targets **PHP 8.3+** and **Laravel 11 / 12 / 13**. Library code only — no application code, no migrations, no commands. Pre-1.0, so backwards-compat breaks are acceptable when called out.

Key non-obvious properties:

- `parent_id` is the source of truth. `fixTree()` rebuilds `lft`/`rgt`/`depth` from a `parent_id` walk.
- Every mutation is one atomic `CASE WHEN UPDATE` (no per-row gap shifts).
- `depth` is **stored and maintained**, not computed at read time.
- Auto-transactions are **on by default** (`config/nestedset.php` → `auto_transaction`).

## Commands

```bash
composer check           # Pint --test → Rector --dry-run → PHPStan, halt on first failure. Run before pushing.
composer test            # PHPUnit, default suite (Package) — excludes Performance + fuzzer group
composer test:parallel   # Same suite under ParaTest (4 workers, ~2.6x on networked engines). Create per-worker DBs first: `DB_CONNECTION=… php tooling/create-worker-databases.php`
composer analyse         # PHPStan / Larastan level 9, NO baseline allowed
composer pint:check      # Laravel Pint, --test mode (style check)
composer rector:check    # Rector --dry-run
composer fuzz            # opt-in seeded fuzzers (PHPUnit --group fuzzer)
composer runabout        # opt-in order-dependent property journeys (PHPUnit --group runabout)
composer test:coverage   # XDEBUG_MODE=coverage phpunit --coverage-text
```

`composer check` is the all-in-one gate — it does **not** run tests, only the static-analysis lane. Run it before every push; pair with `composer test` for full local validation.

Run a single test:

```bash
vendor/bin/phpunit --filter test_method_name
vendor/bin/phpunit tests/Feature/InsertionTest.php
vendor/bin/phpunit testsuite Performance      # benchmarks (opt-in)
```

Backend matrix — set `DB_CONNECTION` to one of `sqlite` (default), `mysql`, `mariadb`, `pgsql`. CI runs every PHP × Laravel × DB cell (PHP 8.3/8.4/8.5 × Laravel 11/12/13 × 4 backends = 36 total).

Fuzzer knobs (`composer fuzz`): `FUZZER_SEEDS`, `FUZZER_STEPS`, `FUZZER_RUNS`, `FUZZER_SEED_COUNT`. Default seeds are pinned to catch known regressions — set `FUZZER_SEEDS=random` for exploration. See `tests/Support/FuzzerConfig.php`.

## Architecture

### The hot path: `NodeTrait`

`src/NodeTrait.php` is the user-facing API entry point. It's a composition of eleven concerns, each owning one slice of behaviour:

| Concern                  | Responsibility                                                                                                                                                                                                                                |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `HasTreeMutation`        | `appendToNode` / `prependToNode` / `insertBeforeNode` / `insertAfterNode` / `makeRoot` / `up` / `down` — queues a `PendingOperation` flushed on `save()`                                                                                      |
| `HasTreeRelations`       | `parent`, `children`, custom `ancestors` / `descendants` eager-loadable relations                                                                                                                                                             |
| `HasTreeRepair`          | `isBroken`, `countErrors`, `fixTree`, `TreeFixResult`                                                                                                                                                                                         |
| `HasSoftDeleteTree`      | cascade soft-delete / restore with `deleted_at` stamp matching                                                                                                                                                                                |
| `HasNodeInspection`      | `isRoot` / `isLeaf` / `isDescendantOf` etc., plus `NodeBounds` value object                                                                                                                                                                   |
| `HasTreeWalk`            | `walk()` / `dfs()` / `dfsPostOrder()` / `bfs()` / `flattenedSubtree()` — in-memory visitor + generators over `relationLoaded('descendants')` or an explicit subtree. Throws `UnloadedSubtreeException` when nothing is loaded; never queries. |
| `HasBulkInsert`          | `bulkInsertTree()` — one `makeGap` + N saves + one deferred `fixAggregates`                                                                                                                                                                   |
| `HasNestedSetAggregates` | precalculated aggregate columns (SUM/COUNT/AVG/MIN/MAX, filtered, listener-based)                                                                                                                                                             |
| `HasTreeExport`          | `toAsciiTree` / `toMermaid` / `toDot` / `toJsonTree` serialisers, plus `*Forest` / `*Scope` static variants, and `fromJsonTree` import                                                                                                        |
| `HasSubtreeClone`        | `cloneSubtreeTo` / `cloneSubtreeAsRoot` — deep-copy a subtree (refresh + delegate to `bulkInsertTree`)                                                                                                                                        |
| `HasMaterialisedPath`    | maintained materialised-path columns (slug/id paths) alongside lft/rgt                                                                                                                                                                        |

Models **must** `implements HasNestedSet` (the contract in `src/Contracts/`). The trait provides default implementations of every interface method.

### Mutations are queued, not immediate

Calls like `appendToNode($parent)` set a `PendingOperation` on the model. The actual lft/rgt manipulation happens on `save()` via the trait's `saving` listener. Pass `->refresh()` parents when staleness matters — the trait re-reads the *target's* bounds before mutating but cannot refresh the node you handed in.

### Query layer

`src/Query/`:

- `TreeQueryBuilder` — extends Eloquent's builder; adds `whereDescendantOf`, `whereAncestorOf`, `whereIsRoot`, `withDepth`, `defaultOrder`, `withFreshAggregates`, etc. Returned automatically from any model using `NodeTrait`.
- `TreeBaseQueryBuilder` — extends Laravel's base `Query\Builder` (a separate hierarchy from `TreeQueryBuilder`). Exists solely to host the MariaDB `SET STATEMENT optimizer_switch='split_materialized=off'` prefix used by the fresh-aggregate read path. Override is consulted in `runSelect()`.
- `TreeMutationBuilder` / `TreeRepairBuilder` — internal builders for the write and repair paths. Tests at `tests/Feature/Query/` exercise these directly.
- `TreeExpression` — thin `Expression` wrapper that lets dynamically composed (but package-owned) SQL bypass Laravel's `@template TValue of literal-string|int|float` constraint. Not a backend dispatcher — the per-driver SQL shapes (LATERAL on PG/MySQL, derived-table on MariaDB, correlated fallback on SQLite) live in `src/Query/Aggregates/Read/AggregateSqlFragments.php` and `FreshAggregateProjector.php`.

### Aggregates subsystem (`src/Aggregates/`)

Three kinds of aggregate column, all sharing the same lifecycle hooks:

- **SQL aggregates** declared via `#[NestedSetAggregate]` — SUM/COUNT/AVG/MIN/MAX over a source column with optional `filter` / `filterNotNull` / `filterRaw` predicates.
- **Listener aggregates** declared via `#[NestedSetAggregateListener]` — PHP `contribution(Model $node)` per row. Supports SUM/COUNT/MIN/MAX/AVG.
- **Ad-hoc fresh aggregates** via `withFreshAggregates(['alias' => Aggregate::sum(...)])` — no column, no maintenance, correlated subquery per outer row.

Maintenance uses two strategies (`Strategy/DeltaMaintenance` vs `Strategy/RecomputeMaintenance`) depending on whether the change can be expressed as a signed delta to ancestors. MIN/MAX and raw-filter columns trigger recompute on the invalidated subset. AVG is auto-promoted to a Sum + Count pair plus the display column.

### Scoping (multi-tree)

`#[NestedSetScope('menu_id')]` or `getScopeAttributes()` declares the partition column(s). `NestedSetScopeResolver` derives the scope; `HasTreeMutation` rejects cross-scope writes with `ScopeViolationException`. Scoped repair methods (`fixTree`, `fixAggregates`, `aggregateErrors`) **require an anchor node** so the repair stays inside one tree.

### Tree diff / JSON import (`src/Diff/`, `src/Import/`)

`TreeDiff::between($a, $b)` diffs two tree snapshots into add/remove/move/modify changes; `->apply($modelClass)` replays them (add → move → remove → modify, one transaction, deferred aggregate pass). `JsonTreeImporter` (behind `Model::fromJsonTree()`) inserts the `toJsonTree()` shape via `bulkInsertTree`. See `docs/querying/tree-diff.md` and `docs/querying/exporters.md`.

### Materialised paths (`src/MaterialisedPath/`)

`HasMaterialisedPath` maintains optional slug/id path columns alongside lft/rgt, rebuilt on the same mutations. See `docs/tree-operations/materialised-paths.md`.

### Service provider

`NestedSetServiceProvider` registers four Blueprint macros: `nestedSet(scope, cover)`, `dropNestedSet`, `nestedSetAggregate(column, type)`, `dropNestedSetAggregate`. Index column order is `scope, lft, rgt, parent_id, ...cover` — scope first so each tree gets its own index slice.

## Code conventions

- **`declare(strict_types=1);`** in every file.
- **PHPStan level 9, no baseline, no `@phpstan-ignore`.** If you need to silence one, fix the type instead.
- **No code comments unless WHY is non-obvious.** The trait files have many comments explaining historical decisions; match that style only when introducing a similar non-obvious choice.
- **Array validation, not pipe strings:** `['required', 'integer']` not `'required|integer'` (Laravel convention from user CLAUDE.md).
- Laravel Pint enforces style. Run `composer pint` to fix.

## Tests

PHPUnit 12 with Orchestra Testbench. Feature tests extend `Vusys\NestedSet\Tests\TestCase`; pure-unit tests that don't need Laravel to boot can extend `PHPUnit\Framework\TestCase` directly (faster, no DB / migrations). `Vusys\NestedSet\Tests\TestCase`:

- Truncates fixture tables in `setUp()` (persistent backends — MySQL/MariaDB/PG — would otherwise leak rows between tests).
- Runs a **tree-integrity check on every fixture in `tearDown()`**. Set `$this->allowBrokenTreeAtTearDown = true` for tests that intentionally leave the tree corrupt (repair tests, force-delete orphan tests).
- Calls `syncSequence($table)` after raw bulk inserts with explicit ids on PostgreSQL — its sequence doesn't auto-advance.

### Fixtures

`tests/Fixtures/Models/` — Category (unscoped), MenuItem (scoped), Area (aggregates), TypedArea, Monster (listener), Branch / SoftBranch / CustomColumnsBranch. **Migration ordering matters** (they're loaded in filename order and TestCase truncates fixed table lists); don't reorder.

Concrete fixture types in test helpers are intentional — they exist so PHPStan can see `NodeTrait` methods without ignore-comments.

### Test categories

- `tests/Unit/` — pure-PHP value-object tests (no DB).
- `tests/Feature/` — DB-backed; one file per concern usually. `Corruption/` covers every corruption category in `docs/maintenance/corruption.md`. Files ending `FuzzerTest.php` are tagged `#[Group('fuzzer')]` and excluded from default runs.
- `tests/Performance/` — separate suite (`vendor/bin/phpunit testsuite Performance`). Cross-backend benchmarks; not run on PR CI.

### Testing helpers

`Vusys\NestedSet\Testing\InteractsWithTrees` exposes `assertIsRoot`, `assertIsLeaf`, `assertIsChildOf`, `assertTreeIsIntact`, `assertAggregatesAreIntact`, etc. Use these in new tests instead of asserting on raw lft/rgt.

## Things to avoid

- Adding `@phpstan-ignore` comments or PHPStan baseline entries — these are explicitly disallowed by `.coderabbit.yaml`.
- Mass-assigning aggregate columns — silently overwritten on next mutation; will produce drift.
- Calling `create()` / `save()` without `appendToNode()` or `makeRoot()` — the saving listener throws `UnplacedNodeException` (an unplaced node can't be persisted through the normal API). Unplaced rows (`lft = rgt = 0`) only arise from raw SQL inserts; `isPlacedInTree()` detects them.
- Reordering existing fixture migrations — `TestCase::setUp` truncates a fixed list.
- Calling `fixTree()` / `fixAggregates()` / `isBroken()` on scoped models without an anchor — throws `ScopeViolationException` by design (prevents accidental full-table walks on multi-million-row forests).
- Adding code comments explaining WHAT — only add when WHY is non-obvious.
- Adding implicit DB hits to the walker — `walk()` deliberately throws `UnloadedSubtreeException` when nothing is loaded. The contract is "respect the caller's load scope"; widening it silently breaks every partial-subtree use case (the test plan asserts no query fires).

## Reference docs

- `README.md` — extensive usage docs; treat as the spec for the public API.
- `docs/maintenance/corruption.md` — corruption taxonomy + recovery recipes.
- `config/nestedset.php` — column names, `auto_transaction`, `aggregate_locking` (`auto`/`always`/`never`), queue routing for `queueFixAggregates`.
