# Model Setup

A `Category` model — the example used throughout these docs — looks like this:

```php
use Illuminate\Database\Eloquent\Model;
use Vusys\NestedSet\Contracts\MaintainsTreeAggregates;
use Vusys\NestedSet\NodeTrait;

class Category extends Model implements MaintainsTreeAggregates
{
    use NodeTrait;

    protected $fillable = ['name'];

    protected $casts = [
        'lft'       => 'integer',
        'rgt'       => 'integer',
        'depth'     => 'integer',
        'parent_id' => 'integer',
    ];
}
```

The trait satisfies the `HasNestedSet` interface (and the `MaintainsTreeAggregates` sub-contract, which adds the aggregate-trait surface) out of the box — you only need to implement methods yourself if you're storing nested-set data on columns the trait can't derive from your `protected $casts`.

## What `NodeTrait` gives you

- Tree mutation primitives: `appendToNode`, `prependToNode`, `insertBeforeNode`, `insertAfterNode`, `makeRoot`, `saveAsRoot`, `up`, `down`. Plus the higher-level `moveTo` / `moveBefore` / `moveAfter` wrappers that pick the right primitive for you. See [Inserting & Moving](../tree-operations/inserting.html).
- Bulk insertion: `bulkInsertTree()` — collapses N saves' worth of gap-shifts into one. See [Bulk Insertion](../tree-operations/bulk-insertion.html).
- Tree-aware query scopes on the model's builder. See [Tree Queries](../querying/queries.html).
- Eloquent relations: `parent`, `children`, `ancestors`, `descendants`. See [Eloquent Relations](../querying/relations.html).
- Inspection: `isRoot`, `isLeaf`, `isChild`, `isDescendantOf`, `isAncestorOf`, `isSiblingOf`, `isPlacedInTree`, `getSubtreeSize`, `getDescendantCount`, plus the read-only sibling lookups `prevSibling` / `nextSibling`. See [Inspection](../querying/inspection.html).
- Repair: `isBroken`, `countErrors`, `fixTree`. See [Tree Repair](../maintenance/fix-tree.html).
- Exporters: `toMermaid`, `toDot`, `toAsciiTree`, `toJsonTree` — read-only formatters for debugging, docs, and frontend handoff, plus the `*Forest` / `*Scope` static counterparts that walk every root in the table. See [Tree Exporters](../querying/exporters.html).
- Optional aggregate maintenance when you declare `#[NestedSetAggregate]` attributes. See [Aggregates Overview](../aggregates/overview.html).
- Typed events fired on Laravel's event bus around every meaningful operation — moves, cascades, repairs, aggregate maintenance, plus model-carrying events for in-process indexing and cache priming. See [Events](../reference/events.html).

## Refresh after mutating a child

> [!WARNING]
> **The #1 footgun.** When you mutate a child under a parent, the parent's in-memory `lft` / `rgt` go stale — the package only refreshes the *target* of the mutation, never the surrounding nodes you already hold references to. Subsequent reads off the stale parent (`->descendants()->get()`, `->getSubtreeSize()`, anything that goes through `getBounds()`) return wrong answers until you `->refresh()` it.
>
> ```php
> $root->saveAsRoot();
> $child->appendToNode($root)->save();
>
> $root->descendants()->get();           // EMPTY — $root->rgt is still 2 in memory
> $root->refresh()->descendants()->get(); // collection containing $child
> ```
>
> The trait reads the *target* node's bounds fresh from the database inside every mutation (the package owns that read), so the tree itself stays safe — but stale references in your code don't. Rule of thumb: if you've held a reference to a parent / sibling across multiple mutations, refresh that reference before reading from it. See [The refresh footgun](../tree-operations/inserting.html#the-refresh-footgun) for the longer explanation.

## Soft deletes

If you add Laravel's `SoftDeletes` trait, the package cascades delete and restore through the subtree. See [Soft Deletes](../tree-operations/soft-deletes.html).

## Scoped (multi-tenant) models

To partition one table into many independent trees — per-tenant menus, per-user folder structures — add `#[NestedSetScope]`. See [Scoped Trees](../querying/scoped-trees.html).

## Exceptions

The trait throws two categories of typed exception. **Programmer-error exceptions** (`\LogicException` subclasses) signal misuse — callers shouldn't try to catch and recover from these in normal flow. **Runtime exceptions** (`\RuntimeException` subclasses) signal data-state problems that may legitimately need handling — corrupted trees, source values outside an aggregate's domain.

### Programmer-error (`LogicException`)

#### `UnplacedNodeException`

`save()` was called on a new node that hasn't been placed in any tree (no `appendToNode` / `prependToNode` / `insertBeforeNode` / `insertAfterNode` / `makeRoot`). Check `$node->isPlacedInTree()` if you need to gate the call.

#### `ScopeViolationException`

A write crossed scope boundaries on a `#[NestedSetScope]` model (`appendToNode($parentInDifferentScope)`), or a scoped model called one of the methods that requires a scoping argument without it:

- `fixTree`, `fixAggregates`, `aggregateErrors` — each requires a `?HasNestedSet $anchor` argument.
- `bulkInsertTree` — requires a `?HasNestedSet $appendTo` second argument. `$appendTo` plays the same anchor role on this method (the inserted rows land under that node and inherit its scope).

The anchor rule prevents accidental full-table walks across millions of rows in many trees. See [Scoped Trees](../querying/scoped-trees.html).

#### `AggregateConfigurationException`

Thrown at registry build time when a `#[NestedSetAggregate]` declaration is malformed (zero or multiple aggregate functions, more than one filter form, empty `filterRawWatches` without the no-column-dependencies opt-in). Boot-time failure by design — surfaces config errors before they become silent drift in production. See [Aggregates → Declaring](../aggregates/declaring.html) and [Filtered Aggregates](../aggregates/filtered.html).

### Runtime (`RuntimeException`)

#### `CorruptTreeException`

The exporters and a few read paths fold the subtree in PHP and refuse to silently infinite-loop on a `parent_id` cycle. Surfaces only on pre-existing corruption (raw `UPDATE` on `parent_id` that bypassed the package); fix with [Tree Repair](../maintenance/fix-tree.html).

#### `AggregateSourceConstraintViolationException`

A save wrote a non-positive value into a `geometricMean` source column, or zero into a `harmonicMean` source column. Either reject the value upstream or declare the aggregate with `allowNonPositive()` to silently skip violating rows instead. See [Geometric & Harmonic Mean](../aggregates/means.html#the-positivity--non-zero-constraint).
