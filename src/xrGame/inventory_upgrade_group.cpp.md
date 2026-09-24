# src/xrGame/inventory_upgrade_group.cpp

> The two structural rules of the upgrade mechanic: you must have installed what unlocks this group, and within a group you may install exactly one.

**Needs** — [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) · [`inventory_upgrade.h`](inventory_upgrade.h.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two list walks over an item's installed-upgrade set

## Purpose

A group is the join in the upgrade forest. Several upgrades may unlock the same group, and a
group holds several mutually exclusive upgrades, so the structure is a directed graph
expressed as alternating layers of nodes and groups rather than a tree.

This file is small and carries one of the mechanic's two real decisions — the other being
the script-code dialect in [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md).

## State

```text
RECORD Group
  id               : text                 # the group's configuration section
  parent_upgrades  : list<UpgradeBase>    # nodes that unlock this group; distinct
  included_upgrades: list<UpgradeBase>    # the mutually exclusive set; authored order
```

**Invariants**
- `included_upgrades` preserves the authored order. The upgrade screen lays a group out in
  that order and the data was drawn against it.
- `parent_upgrades` is distinct by identity. The same node naming a group twice, or two
  paths reaching it, must not make the prerequisite check run twice.
- A root counts as a parent but is never a prerequisite: the root's groups are the ones
  available from the start.

## `construct`

**Contract** — binds the group to its configuration section, records the node that first
mentioned it as a parent, and asks the manager for each upgrade named in the section's
element list. The manager creates an upgrade the first time it is named and returns the
existing one afterwards, which is what lets the graph close.

```text
FUNCTION construct(group_id, parent_upgrade, manager)
  id = group_id                       # section must exist
  add_parent_upgrade(parent_upgrade)
  elements = config id."elements"     # must be non-empty; a group with none is inert
  FOR EACH name IN split_by_comma(elements)
    included_upgrades.append(manager.add_upgrade(name, self))
```

**Notes** — an empty element list is reported and the group is left with no members rather
than failing the process. That is a deliberate softening: a group nobody can pick is a dead
branch in the authored data, not a reason to refuse to start the game.

## `can_install`

**Contract** — asked by a candidate upgrade that belongs to this group, for one item. Returns
`result_e_parents` if something that unlocks this group is missing, `result_e_group` if a
sibling is already installed, and `result_ok` otherwise. During a save restore, either
failure is fatal rather than a refusal, because the save recorded a combination that the
rules say could not have been built.

```text
FUNCTION can_install(item, candidate, loading) -> UpgradeStateResult
  FOR EACH parent IN parent_upgrades
    IF parent.is_root()
      CONTINUE                        # the root unlocks its groups unconditionally
    IF game IS clear_sky
      missing = NOT item.has_upgrade(parent.id)
    ELSE
      missing = NOT item.has_upgrade_group(parent.parent_group_id)
    IF missing
      IF loading THEN FAIL WITH "save records an unreachable upgrade combination"
      RETURN result_e_parents

  FOR EACH sibling IN included_upgrades
    IF sibling IS candidate
      CONTINUE
    IF item.has_upgrade(sibling.id)
      IF loading THEN FAIL WITH "save records two upgrades from one exclusive group"
      RETURN result_e_group

  RETURN result_ok
```

**Notes** — the prerequisite test differs between games and this is the decision worth
carrying. *Clear Sky* requires **that exact parent upgrade** to be installed. The two newer
games require only that **some upgrade from the parent's group** is installed — meaning any
of the mutually exclusive alternatives at the previous tier satisfies the next tier, which
is what makes their upgrade trees branchable rather than a single chain. The engine branches
on the game identity flag because the shipped data for each game was authored against its
own rule and neither reading can be inferred from the data itself. The source marks this as
wanting a data-driven solution; none exists.

Treating a `loading` failure as fatal rather than as a dropped upgrade is also a decision:
silently discarding an upgrade during a restore would leave the player's item quietly weaker
than it was when they saved, which is worse than refusing the save.

## `highlight_up` / `highlight_down`

**Contract** — the two directions of the screen's dependency highlight. `highlight_up`
pushes forward into every included upgrade; `highlight_down` pushes backward into every
parent. A group has no flag of its own — it only relays.

## `fill_root` / `add_parent_upgrade` / `id`

**Contract** — `fill_root` asks each included upgrade to register itself into the given
root's flat lookup and recurse. `add_parent_upgrade` attaches a node that unlocks this
group, ignoring a repeat. `id` is the group's configuration section name.
