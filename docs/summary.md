# Summary

This file is the source of truth for the docs site navigation. The build script reads it to build the sidebar and prev/next links.

Format:

- A top-level `#` is a section title.
- A list item is a page: `- [Title](path/to/file.md)`.
- The first page in the first section is the home page (`index.html`).
- Files that don't yet exist render as placeholders so the structure can be sketched up-front and filled in incrementally.

# Getting Started

- [Introduction](index.md)
- [Installation](getting-started/installation.md)
- [Migration](getting-started/migration.md)
- [Primary Keys](getting-started/primary-keys.md)
- [Model Setup](getting-started/model-setup.md)

# Tree Operations

- [Inserting & Moving](tree-operations/inserting.md)
- [Reordering Siblings](tree-operations/reordering.md)
- [Soft Deletes](tree-operations/soft-deletes.md)
- [Bulk Insertion](tree-operations/bulk-insertion.md)
- [Cloning Subtrees](tree-operations/cloning.md)
- [Materialised Paths](tree-operations/materialised-paths.md)

# Querying

- [Tree Queries](querying/queries.md)
- [Eloquent Relations](querying/relations.md)
- [In-memory Tree Shaping](querying/tree-shaping.md)
- [Walking Subtrees](querying/walking.md)
- [Tree Exporters](querying/exporters.md)
- [Tree Diff](querying/tree-diff.md)
- [Inspection](querying/inspection.md)
- [Scoped Trees](querying/scoped-trees.md)

# Aggregates

- [Overview](aggregates/overview.md)
- [Migration & Setup](aggregates/setup.md)
- [Reading Values](aggregates/reading.md)
- [Declaring Aggregates](aggregates/declaring.md)
- [Filtered Aggregates](aggregates/filtered.md)
- [Collection Aggregates](aggregates/text-and-json.md)
- [Listener Aggregates](aggregates/listeners.md)
- [Variance & Stddev](aggregates/maths.md)
- [Weighted Average & Boolean Rollups](aggregates/weighted-avg-and-booleans.md)
- [Geometric & Harmonic Mean](aggregates/means.md)
- [Median, Percentile & Quantiles](aggregates/quantiles.md)
- [Top-K Aggregate](aggregates/top-k.md)
- [Bitwise Aggregates](aggregates/bitwise.md)
- [Lazy Aggregates](aggregates/lazy.md)
- [Recipes](aggregates/recipes.md)
- [Maintenance](aggregates/maintenance.md)
- [Drift & Limitations](aggregates/drift.md)

# Maintenance

- [Tree Repair](maintenance/fix-tree.md)
- [Repairing Aggregates](maintenance/fix-aggregates.md)
- [Corruption Reference](maintenance/corruption.md)

# Reference

- [Configuration](reference/config.md)
- [Testing Helpers](reference/testing.md)
- [Factory Tree Builder](reference/factories.md)
- [Transactions](reference/transactions.md)
- [Events](reference/events.md)
- [Production Notes](reference/production.md)
- [Glossary](reference/glossary.md)

# Internals

- [Architecture Overview](internals/architecture.md)
- [The Nested-Set Model](internals/nested-set-model.md)
- [The Mutation Engine](internals/mutation-engine.md)
- [Query Engine & Relations](internals/query-engine.md)
- [Aggregate Maintenance](internals/aggregate-maintenance.md)
- [Integrity & Repair](internals/repair.md)
- [Concurrency & Transactions](internals/concurrency.md)
- [The Test Suite](internals/test-suite.md)
