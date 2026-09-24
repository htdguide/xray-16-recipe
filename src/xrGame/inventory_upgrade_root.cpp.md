# src/xrGame/inventory_upgrade_root.cpp

> The entry point of one item's upgrade tree: it reads the item's upgrade declaration, keeps a flat index of everything below it, and resolves the upgrade screen's grid cells.

**Needs** — [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md) · [`inventory_upgrade.h`](inventory_upgrade.h.md) · [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) · [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`game_type.h`](game_type.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: walks an object graph built from configuration

## Purpose

The upgrade forest is a tree of groups and upgrades, but two questions about it are asked
constantly and cannot be answered by walking: "does this item have this upgrade" and "which
upgrade is in grid cell (x, y)". The root keeps a **flat list of every upgrade beneath it**,
filled during construction, so both are a linear scan over one vector instead of a
recursive descent. That flat list, plus the item's layout scheme name, is the whole of what
a root adds to the shared node type.

## State

```text
RECORD Root EXTENDS UpgradeNode
  scheme    : text            # names the screen layout authored for this item
  contained : list<Upgrade>   # EVERY upgrade beneath this root, flattened, borrowed
```

**Invariant** — a root is *known* from the moment it is constructed. Roots are not things
the player discovers; only the upgrades beneath them are.

**Invariant** — `contained` holds no duplicates, and the tree beneath the root remains the
authority. The flat list is an index, not an ownership structure: the nodes are owned by
[`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md).

**Invariant** — scheme indices are unique within one root. Two upgrades of the same item
claiming the same grid cell means one of them is unreachable from the screen. This is
checked and *reported* as each upgrade registers, never enforced — the shipped data is
clean and a mod that breaks it still runs, with one cell shadowing the other.

## `construct`

**Contract** — reads the item's upgrade declaration and builds the tree beneath it. An item
section with no `upgrades` key, or an empty one, produces an empty root rather than a
failure — the manager's discovery scan should not have offered it, but the check is cheap
and keeps a partial data edit from aborting the game. Recursive: creating the dependent
groups creates their upgrades, which create the groups they unlock, and so on.

```text
FUNCTION construct(item_section, manager)
  construct as a shared node; mark known
  IF the section has no non-empty "upgrades" THEN RETURN   # empty root
  add_dependent_groups(section.upgrades, manager)          # recurses the whole subtree
  scheme = section.upgrade_scheme                          # warned about, not required
  fill_root_container(this)                                # tells every descendant its root
```

**Invariant** — the flat list is filled *after* the subtree exists, by a downward walk that
hands each descendant a reference to this root; each upgrade then registers itself back via
`add_upgrade`. The two-phase shape is necessary because an upgrade cannot know its root
while the root is still being constructed.

## `add_upgrade`

**Contract** — registers one descendant upgrade into the flat list, ignoring a repeat —
which happens legitimately, because a group reachable from two branches is walked twice.
Reports a duplicated scheme index as it goes, in a non-shipping build.

## `contain_upgrade`

**Contract** — whether a named upgrade belongs to this item's tree. Asks the shared node's
own answer first (the direct children), then each entry of the flat list — which itself
recurses into that upgrade's unlocked subtree. Total; absent is a plain negative.

**Notes** — because the flat list already holds every descendant, the recursion through
each entry is redundant work, not a correctness requirement. A rebuild that maintains the
flat list correctly can answer this with a set membership test.

## `verify_scheme_index` · `get_upgrade_by_index`

**Contract** — the two directions of the grid mapping. `verify_scheme_index` answers
whether a cell is still *free*, which is only used to diagnose authoring collisions;
`get_upgrade_by_index` returns the upgrade occupying a cell, or nothing. Both are linear
scans of the flat list. The grid is a few dozen cells at most, so a rebuild gains nothing
by indexing it by cell.

## `highlight_hierarchy` · `reset_highlight`

**Contract** — lights the tree path associated with one named upgrade, and clears all
lighting. Finds the named upgrade in the flat list and lights **downward** — the upgrade
and everything it unlocks — and, *in Clear Sky only*, **upward** as well: the prerequisite
chain that leads to it.

**Notes** — the direction of the highlight is the only per-game branch in the upgrade
system, and it reflects a difference in how the two games' upgrade screens read. Clear Sky
shows a branching tree where the player needs to see the cost of a whole path; Call of
Pripyat shows a grid where only the consequences matter. The original flags this as a place
wanting a data-driven answer rather than a game check, and a rebuild should make the
direction a property of the layout scheme.

## `is_root`

**Contract** — identifies this node as a root, where the shared node type answers false.
This is how the upward walk from an upgrade knows it has arrived, and it is a virtual
answer rather than a type test because the walk holds the shared type.

## `log_hierarchy` · `test_all_upgrades`

**Contract** — development-only diagnostics. The first prints the subtree with indentation;
the second asks the item to validate every upgrade section beneath this root — that is,
that each one's configuration merges cleanly — and reports each result. `test_all_upgrades`
runs at every item's first install attempt in a development build, which is what catches a
mistyped upgrade section before a player pays for it.

**Notes** — the diagnostics are gated behind a console-settable verbosity level shared with
[`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md); it defaults to silent.
