# src/xrGame/inventory_upgrade_base.cpp

> The two checks every upgrade node performs before anything item-specific is considered: has the player learned of it, and is it already installed.

**Needs** — [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) · [`inventory_upgrade.h`](inventory_upgrade.h.md)
**Used by** — [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md)
**Tier floor** — T3: list walking and two predicates

## Purpose

The base half of an upgrade-forest node. It exists as a separate file only because a root
and an upgrade share these behaviours; there is nothing a rebuilder must keep separate
here, and folding it into the two derived types is legitimate.

## State

Owns the node's identifier, its known flag and its list of dependent groups — declared in
[`inventory_upgrade_base.h`](inventory_upgrade_base.h.md).

## `construct`

**Contract** — records the node's identifier and starts it *unknown*. Does not read the
node's configuration section; each derived kind does that itself, because a root and an
upgrade need different keys from it.

**Notes** — the development-build diagnostic here fires when the section *does* exist, which
is inverted with respect to its message ("section not exist"). Nothing depends on it; it is
a bug in a warning, and a rebuild should emit the warning on the absent case.

## `add_dependent_groups`

**Contract** — takes a comma-separated list of group names, asks the manager for each group
(creating it on first mention), and attaches it to this node. Duplicates in the authored
list are silently collapsed — the same group may legitimately be named by several upgrades,
and attaching it twice would make the highlight walk visit it twice.

```text
FUNCTION add_dependent_groups(groups_text, manager)
  FOR EACH name IN split_by_comma(groups_text)
    group = manager.add_group(name, self)     # creates it if this is its first mention
    IF group NOT IN depended_groups
      depended_groups.append(group)
```

## `can_install`

**Contract** — the two universal refusals. Returns a verdict; never mutates.

```text
FUNCTION can_install(item, loading) -> UpgradeStateResult
  # while restoring a save the tree is replayed in an order that has already
  # been validated once, so the unknown check would fire spuriously
  IF NOT known AND NOT loading
    RETURN result_e_unknown
  IF item.has_upgrade(id)
    RETURN result_e_installed
  RETURN result_ok
```

**Notes** — the `loading` flag runs through the entire upgrade mechanic and means "we are
reconstructing state that was already accepted once". Every check that consults the player's
knowledge or the world's state is skipped under it; every check that would indicate a
*corrupt* save is escalated from a refusal to a fatal error in the derived types. That split
— skip the policy checks, harden the consistency checks — is the design decision worth
carrying over.

## `fill_root_container`

**Contract** — recursive collection. Asks each dependent group to pour its own upgrades into
the given root's flat container. The forest is authored as a graph of names; the root needs
a flat list of every node reachable from it so that a lookup by upgrade name does not have
to walk.

## `is_root` / `make_known` / `contain_upgrade`

**Contract** — `is_root` is false here and overridden by the root node. `make_known` sets the
learned flag and always reports success. `contain_upgrade` compares this node's identifier
against the given one; the root overrides it with a container lookup.

**Notes** — identifier comparison is by *interned string identity*, not by text. Every
upgrade name in the system comes from the shared string pool, so equality is a pointer
compare and the forest can be walked cheaply. A rebuild either interns names the same way or
accepts the text comparison.

## `log_hierarchy`

**Contract** — development-build only: prints the subtree under this node, indented. A
diagnostic, not a mechanism.
