# laravel-nestedset docs site

A small, dependency-light static site generator for the project documentation. Pure PHP. Reads Markdown from `../docs`, writes HTML to `../site`. Third-party CSS and JS (normalize.css, Prism) are loaded from a CDN at runtime, so the only build dep is `league/commonmark`.

## One-time setup

```bash
cd docs-site
composer install
```

## Build the site

```bash
composer build
# or: php build.php
```

Output goes to `../site/`. It's safe to delete — `composer clean` does it for you.

## Preview locally with live reload

```bash
composer serve
# or: php serve.php --port=8000
```

This:

1. Builds the site once.
2. Starts PHP's built-in dev server on [http://localhost:8000](http://localhost:8000).
3. Watches `docs/` and `docs-site/` (templates, CSS, JS, `build.php`) for changes and rebuilds on save. Your browser auto-reloads.

Pass `--no-watch` to skip the watcher and just serve. Pass `--port=N` to use a different port.

## Authoring

- Pages live in `../docs/` as plain Markdown.
- Navigation is defined in `../docs/summary.md` — top-level `#` headings are sections, list items are pages: `- [Title](path/to/file.md)`. Add a new page by adding a line there.
- Pages listed in `summary.md` that don't yet exist render as placeholders, so you can plan the site upfront and fill it in incrementally.

## Layout

```text
docs-site/
├── build.php           # Markdown → HTML, applies layout, expands callouts
├── serve.php           # Build + dev server + file watcher
├── composer.json       # league/commonmark only
├── templates/
│   └── layout.php      # Single page template — links normalize.css and
│                       # Prism from jsdelivr CDN
└── public/
    ├── style.css       # All site styles
    ├── app.js          # Theme toggle, code copy buttons, TOC scroll spy, live reload
    ├── tree-widget.css # Styles for the nested-set tree widget
    └── tree-widget.js  # The nested-set tree widget
```

Output lives in `../site/` (gitignored). Static files in `public/` are copied as-is at build time.

## Callouts

Author GitHub-style alerts in any Markdown page; `build.php` expands them into styled callouts whose body is still rendered as Markdown:

```markdown
> [!NOTE]
> Body text, with **Markdown** and [links](other.md).
```

Supported: `NOTE`, `TIP`, `IMPORTANT`, `WARNING`, `CAUTION`. An optional title may follow the marker: `> [!TIP] Heads up`.

## Tree widget

Drop an indented tree into a fenced `ns-tree` block; indentation is the hierarchy. `lft`/`rgt`/`depth` are computed for you. A numeric brace annotation becomes an aggregate value that rolls up every ancestor as a SUM; other key/value pairs render as chips:

````markdown
```ns-tree
Electronics
  Phones
    Android {products=37}
    iOS {products=15}
```
````

Selecting a node highlights its subtree + ancestors and shows the `BETWEEN` query (and matching `SUM` when a metric is present).

**Rewrite-ready data contract.** `tree-widget.js` is split into a JS authoring adapter (`NestedTree.fromText`) and a source-agnostic renderer (`NestedTree.render(mountEl, data)`), both on `window.NestedTree`. The renderer consumes a flat node list — `{ metric, nodes: [{ id, parentId, name, lft, rgt, depth, value, rollup, chips }] }` — so a future package-driven Laravel app can produce the identical shape from `Model::defaultOrder()->get()` (the maintained `<name>_total` column is each row's `rollup`) and reuse the same renderer over server-side AJAX. Partial-subtree fetches work too: any node whose `parentId` is absent from the set is rendered as a root.
