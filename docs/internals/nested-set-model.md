# The Nested-Set Model

Everything else in the package rests on one encoding: each node stores two integers, `lft` and `rgt`, that bracket its entire subtree. This page covers the encoding, the invariants the code relies on, and the value object that carries the bounds around.

## Four columns

`$table->nestedSet()` adds four columns to your table. The default names live in `src/Columns.php`:

```php
final class Columns
{
    const string LFT = 'lft';
    const string RGT = 'rgt';
    const string PARENT_ID = 'parent_id';
    const string DEPTH = 'depth';
    // ...
}
```

| Column      | Type                            | Role                                                              |
| ----------- | ------------------------------- | ----------------------------------------------------------------- |
| `lft`       | unsigned bigint                 | Opening boundary of the node's subtree in a pre-order walk.       |
| `rgt`       | unsigned bigint                 | Closing boundary. Always `> lft`.                                 |
| `parent_id` | nullable (matches the key type) | **The source of truth** for parent/child links. `NULL` for roots. |
| `depth`     | unsigned int                    | Level in the tree (roots at 0). Stored, not computed.             |

The names are read through config-backed accessors on the model (`getLftName()` etc. in `src/NodeTrait.php`), so a project can rename them once in `config/nestedset.php` without touching the package.

## The encoding, by example

Imagine a pre-order walk of the tree: as you enter a node you stamp `lft` from a running counter, descend into its children, and on the way back out you stamp `rgt`. Every node's `[lft, rgt]` interval therefore *contains* the intervals of all its descendants. Take this small category tree:

```text
Electronics
├── Computers
│   ├── Laptops
│   └── Desktops
└── Phones
    ├── Android
    └── iOS
Clothing
├── Shoes
└── Outerwear
```

Numbered out, each node's interval wraps its whole subtree — that containment is the entire trick. The `Electronics` subtree looks like this:

```text
              1            14
        ┌──── Electronics ────┐
        │                     │
     2     7              8       13
   Computers              Phones
   ┌──┴──┐               ┌──┴──┐
  3 4   5 6            9 10  11 12
 Laptops Desktops     Android  iOS
```

`Electronics` spans `1..14`; everything inside that interval is a descendant. `Computers` spans `2..7`, so `Laptops` (`3..4`) and `Desktops` (`5..6`) are its descendants — their intervals nest inside `2..7`. A leaf like `Laptops` occupies two adjacent slots (`3` and `4`).

## The invariants

The whole engine — mutation, querying, and repair — is written to preserve a short list of invariants. The repair tooling ([Integrity & Repair](repair.html)) is literally defined as "detect and fix violations of these":

- **`lft < rgt`** for every node. A row where `lft >= rgt` is the `invalid_bounds` corruption.
- **A leaf has `rgt = lft + 1`** — no gap between its own boundaries.
- **A subtree of N nodes (counting itself) occupies exactly 2N slots**, so the count of strict descendants is `(rgt - lft - 1) / 2`.
- **`lft` and `rgt` values are each unique within a scope** — collisions are the `duplicate_lft` / `duplicate_rgt` corruptions.
- **Containment matches ancestry**: `A` is an ancestor of `B` iff `A.lft < B.lft AND B.rgt < A.rgt`.
- **Per scope, the bounds form a contiguous `1..2N` permutation** — no holes. Mutations that remove a node close the gap they leave behind so this stays true.

These show up directly in the inspection predicates (`src/Concerns/HasNodeInspection.php`), which compute everything from the in-memory bounds without touching the database:

```php
public function isRoot(): bool
{
    return $this->getParentId() === null;
}

public function isLeaf(): bool
{
    return $this->getRgt() - $this->getLft() === 1;
}

public function getSubtreeSize(): int
{
    return $this->getRgt() - $this->getLft() + 1;
}

public function getDescendantCount(): int
{
    return (int) (($this->getRgt() - $this->getLft() - 1) / 2);
}
```

> [!NOTE]
> `getSubtreeSize()` returns the *slot* count (`rgt - lft + 1` = 2N), not the node count. That is exactly the gap width a bulk insert needs to reserve, so the slot count is the more useful primitive internally. For the count of descendants, use `getDescendantCount()`.

## `NodeBounds` — the bounds value object

Rather than pass `lft`/`rgt`/`depth` around as loose integers, the package wraps them in an immutable value object, `NodeBounds` (`src/NodeBounds.php`). It is tiny, and it encodes the containment relation directly:

```php
readonly class NodeBounds
{
    public function __construct(
        public int $lft,
        public int $rgt,
        public int $depth,
    ) {}

    public function height(): int
    {
        return $this->rgt - $this->lft + 1;
    }

    /** True when $other is strictly inside this node's bounds. */
    public function contains(self $other): bool
    {
        return $this->lft < $other->lft && $other->rgt < $this->rgt;
    }

    /** Depth change going from $this to $other. */
    public function depthDelta(self $other): int
    {
        return $other->depth - $this->depth;
    }
}
```

`contains()` *is* the descendant test — `isDescendantOf()` and `isAncestorOf()` are one-liners over it:

```php
public function isDescendantOf(HasNestedSet $other): bool
{
    return $other->getBounds()->contains($this->getBounds());
}
```

A model produces its bounds through the `HasNestedSet` contract method `getBounds()`, implemented on `NodeTrait`:

```php
public function getBounds(): NodeBounds
{
    return new NodeBounds(
        lft: $this->getLft(),
        rgt: $this->getRgt(),
        depth: $this->getDepth(),
    );
}
```

`NodeBounds` is the currency of the mutation and aggregate engines: the move algorithm takes a `from` bounds and a target position; the aggregate hooks take `from`/`to` bounds to find the old and new ancestor chains. Because it is `readonly`, a captured `from` snapshot stays valid even after the underlying row has been renumbered by the SQL.

## Why `parent_id` is the source of truth {#parent-id-source-of-truth}

`lft`/`rgt`/`depth` are a fast **index** over the hierarchy — they make any subtree, ancestor chain, or descendant set a single `BETWEEN` query. But they are also fragile: a half-applied move, an out-of-band `UPDATE`, or a crashed transaction can leave them inconsistent.

`parent_id` is different. It is a single, local fact per row — "who is my parent" — that no tree-wide arithmetic depends on. So the package treats it as canonical: if the bounds and `parent_id` ever disagree, `parent_id` wins, and `fixTree()` rebuilds `lft`/`rgt`/`depth` from a `parent_id` walk ([Integrity & Repair](repair.html)). This is why the package maintains `parent_id` on every mutation even though pure nested-set theory does not require it — it is the recovery anchor.

> [!TIP]
> The corollary: never write `lft`/`rgt` by hand or mass-assign them. Treat them as engine-owned. If you need to reshape the tree, change `parent_id` (via the [mutation API](mutation-engine.html)) and let the engine renumber.

## Where to go next

[The Mutation Engine](mutation-engine.html) shows how inserts and moves keep all of these invariants true in a single statement.
