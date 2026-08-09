# Contributing

Thanks for considering a contribution. This package is library code only (no application scaffolding, no migrations, no commands) — issues and PRs are most useful when they ship with tests against the package's own suite.

## Quick start

```bash
composer install
composer test          # default suite on sqlite
composer analyse       # PHPStan level 9, no baseline allowed
composer pint:check    # Laravel Pint style check
composer rector:check  # Rector dry-run
```

Open issues live at [https://github.com/Vusys/laravel-nestedset/issues](https://github.com/Vusys/laravel-nestedset/issues). Pre-1.0, so backwards-compat breaks are acceptable when called out.

## Running tests across backends

CI runs the full 3×3×4 matrix (PHP × Laravel × DB) on every PR. Locally, the `composer test:matrix` helper runs the default suite against each supported backend sequentially and halts on the first red cell:

```bash
composer test:matrix
```

That's equivalent to the loop below, but handles the env-var threading for you:

```bash
DB_CONNECTION=sqlite  composer test
DB_CONNECTION=mysql   composer test
DB_CONNECTION=mariadb composer test
DB_CONNECTION=pgsql   composer test
```

Single-backend runs work the same way — pass `DB_CONNECTION` ad-hoc:

```bash
DB_CONNECTION=pgsql composer test
```

### Backend prerequisites

`sqlite` runs in-memory and needs nothing. The networked backends expect a running server reachable on `127.0.0.1` with the defaults wired in `tests/TestCase.php::defineEnvironment()`:

| Backend         | Port | Database | User     | Password |
| --------------- | ---- | -------- | -------- | -------- |
| mysql / mariadb | 3306 | testing  | root     | password |
| pgsql           | 5432 | testing  | postgres | password |

Override any of these via the standard `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` env vars before invoking the test command.

## Other useful commands

```bash
composer fuzz                    # seeded fuzzers (opt-in, --group fuzzer)
composer test:coverage           # XDEBUG_MODE=coverage phpunit --coverage-text
composer infection               # mutation testing on the current backend
vendor/bin/phpunit --filter <name>           # one test by name
vendor/bin/phpunit tests/Feature/<File>.php  # one file
vendor/bin/phpunit testsuite Performance     # opt-in benchmark suite
```

See [`CLAUDE.md`](CLAUDE.md) for the project-level architecture notes.

## Changelog

User-facing changes go in [`CHANGELOG.md`](CHANGELOG.md) under **Unreleased** (Keep a Changelog format). Call out any pre-1.0 backwards-compatibility break under **Changed** / **Removed**.
