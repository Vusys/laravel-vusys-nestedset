# The Test Suite

This package keeps **derived state** — `lft` / `rgt` / `depth`, aggregate columns, materialised paths — in sync with `parent_id` through single atomic `CASE WHEN` statements, on four database engines, under arbitrary orderings of mutations. There is no runtime type system that can prove that correct, so it is defended by breadth of tests instead. This page is a map of that suite: what the categories are, what each defends, and how to run them.

> [!NOTE]
> The counts below are a snapshot (captured on the `runabout-tests` branch) and drift as the suite grows — treat them as orders of magnitude, not contract. The commands and structure are the durable part. Regenerate any number with the one-liners in [Counting for yourself](#counting-for-yourself).

## By the numbers

The default suite (`composer test`, sqlite) runs roughly:

| Metric                                             | Count                             |
| -------------------------------------------------- | --------------------------------- |
| Test cases executed                                | \~2,260                           |
| `#[Test]` methods (before data-provider expansion) | \~1,850                           |
| Assertions                                         | \~23,000                          |
| Test files (`*Test.php`)                           | \~280                             |
| Fixture models (`tests/Fixtures/Models/`)          | 43                                |
| Database engines exercised                         | 4 (sqlite, mysql, mariadb, pgsql) |
| Full CI matrix cells                               | 36 (3 PHP × 3 Laravel × 4 DB)     |

The default run **excludes** the opt-in fuzzer and runabout groups and the Performance suite; those add substantially more coverage on demand.

## Layout

```
tests/
├── Unit/          pure value-object tests — no DB, no Laravel boot
├── Feature/       DB-backed, one concern per subdirectory
├── Runabout/      order-dependent property journeys (opt-in group)
├── Performance/   cross-backend benchmarks (separate suite)
├── Fixtures/      43 models + factories + migrations
├── Support/       FuzzerConfig and other shared test plumbing
└── TestCase.php   the base class every DB-backed test extends
```

`tests/Feature/` is the bulk of the suite, split one directory per concern:

`Aggregates`, `BulkInsert`, `Clone`, `Concurrency`, `Corruption`, `Diff`, `Events`, `Export`, `Fuzzers`, `Import`, `Inspection`, `MaterialisedPath`, `Mutation`, `Query`, `Relations`, `Schema`, `Scoping`, `SoftDelete`, `Testing`, `Walk`.

## The categories

### Unit (`tests/Unit/`)

Pure-PHP tests of value objects and algorithms that need neither a database nor a booted Laravel container — `NodeBounds`, `WalkContext`, aggregate math, path builders. They extend `PHPUnit\Framework\TestCase` directly, so they are fast and run without migrations.

### Feature (`tests/Feature/`)

DB-backed tests that boot Orchestra Testbench and extend `Vusys\NestedSet\Tests\TestCase`. Each subdirectory owns one slice of the public API. Two are worth calling out:

- **`Corruption/`** walks the corruption taxonomy in [Corruption Reference](../maintenance/corruption.html) — one repair scenario per category (invalid bounds, orphans, duplicate `lft`, depth mismatch, …), seeding a broken tree via raw SQL and asserting `fixTree()` recovers it.
- **`Fuzzers/`** (`*FuzzerTest.php`, `#[Group('fuzzer')]`) drive random mutation sequences against an oracle. They are **excluded from the default run** and triggered with `composer fuzz`. Seeds are pinned by default so a known regression re-triggers deterministically; set `FUZZER_SEEDS=random` to explore. Knobs live in `tests/Support/FuzzerConfig.php`: `FUZZER_SEEDS`, `FUZZER_STEPS`, `FUZZER_RUNS`, `FUZZER_SEED_COUNT`.

### Runabout journeys (`tests/Runabout/`)

The [Runabout](https://github.com/Vusys/laravel-runabout) journeys are property tests for **order-dependent** invariants: each journey declares weighted, repeatable mutation steps plus standing invariants, and the framework runs them in seeded shuffles, checking every invariant after every step and **shrinking** any failing order to a minimal reproduction. They are tagged `#[Group('runabout')]` and excluded from the default run.

Ten journeys, one per order-sensitive concern — SQL and listener aggregates, soft-delete lifecycle, multi-tree scope, materialised paths, subtree clone, two flavours of tree-diff apply, `fixTree` repair convergence, and JSON round-trip. Each is double-guarded: the library's own recompute (`aggregateErrors()` / `countErrors()`) **plus** an independent from-scratch oracle with no package code in the loop. This suite found and fixed two real `TreeDiff::apply()` corruption bugs; see [Tree Diff](../querying/tree-diff.html).

### Performance (`tests/Performance/`)

A separate PHPUnit suite of cross-backend benchmarks (insert/move/repair at scale). Not part of PR CI; run explicitly with `vendor/bin/phpunit --testsuite Performance`.

## The harness: `TestCase`

Every DB-backed test inherits three safety behaviours from `Vusys\NestedSet\Tests\TestCase` (`tests/TestCase.php`):

- **Truncation in `setUp()`** — persistent backends (MySQL/MariaDB/PostgreSQL) would otherwise leak rows between tests, so a fixed list of fixture tables is truncated before each test.
- **A tree-integrity check in `tearDown()`** — after *every* test, each fixture tree is validated for corruption. A test that intentionally leaves a tree broken (repair tests, forced-orphan tests) must opt out with `protected bool $allowBrokenTreeAtTearDown = true;`. This is the net that catches a mutation which passes its own assertions but silently corrupts the tree.
- **`syncSequence()` on PostgreSQL** — after raw bulk inserts with explicit ids, Postgres' sequence does not auto-advance, so the harness resyncs it.

User-facing assertion helpers (`assertIsRoot`, `assertIsLeaf`, `assertTreeIsIntact`, `assertAggregatesAreIntact`, …) live in `Vusys\NestedSet\Testing\InteractsWithTrees` and are documented for **your** tests in [Testing Helpers](../reference/testing.html).

## Running the tests

| Command                                        | What it runs                                                                                        |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `composer test`                                | Default `Package` suite (sqlite) — excludes Performance, fuzzer, runabout                           |
| `composer check`                               | Static gate: Pint (style) → Rector (dry-run) → PHPStan level 9, no baseline. Halts on first failure |
| `composer test:coverage`                       | Package suite with text coverage (`XDEBUG_MODE=coverage`)                                           |
| `composer test:matrix`                         | Package suite once per backend (sqlite → mysql → mariadb → pgsql)                                   |
| `composer fuzz`                                | The `fuzzer` group only                                                                             |
| `composer runabout`                            | The `runabout` group only (order-dependent property journeys)                                       |
| `composer infection`                           | Mutation testing (Infection)                                                                        |
| `vendor/bin/phpunit --testsuite Performance`   | The benchmark suite                                                                                 |
| `vendor/bin/phpunit --filter test_method_name` | A single test                                                                                       |
| `vendor/bin/phpunit tests/Feature/Diff`        | One directory                                                                                       |

`composer check` is the static-analysis lane only — it does **not** run tests. Pair it with `composer test` before pushing.

### Choosing a backend

Set `DB_CONNECTION` to `sqlite` (default), `mysql`, `mariadb`, or `pgsql`. The non-sqlite engines need a running server; `tooling/db-compose.yml` brings them up locally (`docker compose -f tooling/db-compose.yml up`). Some classes of bug — `deleted_at` sub-second precision, foreign-key ordering — only surface off sqlite, which is why the matrix exists.

### Counting for yourself

```bash
# test files and #[Test] methods
find tests -name '*Test.php' | wc -l
grep -rho '#\[Test\]' tests | wc -l

# opt-in group members
grep -rl "Group('fuzzer')"  tests | wc -l
grep -rl "Group('runabout')" tests | wc -l

# full run with the live totals in the footer
composer test
```

## The CI matrix

`.github/workflows/tests.yml` is a staged, fail-fast pipeline that gates the expensive matrix on cheap deterministic checks:

```
tier 0  static-analysis + actionlint
tier 1  test-13  ‖  runabout  ‖  fuzz-pinned   (cross-backend, pinned seeds)
tier 2  test-legacy (Laravel 11 + 12)
```

The full grid is **PHP 8.3 / 8.4 / 8.5 × Laravel 11 / 12 / 13 × {sqlite, mysql, mariadb, pgsql}** = 36 cells. Runabout and the fuzzers run with **pinned seeds** in CI so they are deterministic and safe as required gates; random-seed exploration stays on a separate weekly cron. Because the tier-1 jobs finish inside `test-13`'s wall-clock, they add near-zero latency but block the legacy matrix the moment they regress.

## What guards what

| If you change…                                        | The suite that will catch a regression                                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Mutation SQL (`appendToNode`, `moveTo`, gap shifting) | `Feature/Mutation`, `Feature/Corruption`, `TreeStructureFuzzer`, every runabout journey's structure invariant |
| Aggregate maintenance                                 | `Feature/Aggregates`, `AggregateFuzzer`, `AreaAggregateJourney` / `MonsterHordeJourney`                       |
| `fixTree` / repair                                    | `Feature/Corruption`, `RepairConvergenceJourney`                                                              |
| Tree diff apply                                       | `Feature/Diff`, `DiffRoundTripJourney`, `DiffAddRemoveJourney`                                                |
| Bulk insert / clone / JSON import                     | `Feature/BulkInsert`, `Feature/Clone`, `Feature/Import`, `CloneOrchardJourney`, `JsonRoundTripJourney`        |
| Anything touching the tree at all                     | `TestCase`'s `tearDown()` integrity check, on every DB-backed test                                            |

See also: [Integrity & Repair](repair.html) for what `fixTree` guarantees, and [Testing Helpers](../reference/testing.html) for the assertions you can reuse in your own tests.
