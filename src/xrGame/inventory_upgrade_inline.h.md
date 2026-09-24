# src/xrGame/inventory_upgrade_inline.h

> The read-only accessors of one upgrade node: its configuration section, its parent group, and its presentation fields.

**Needs** — [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) · [`inventory_upgrade.h`](inventory_upgrade.h.md)
**Used by** — [`inventory_upgrade.h`](inventory_upgrade.h.md)
**Tier floor** — T3: field reads

## Purpose

Splits the trivial accessors of the upgrade node out of its declaration so that the
declaration reads as a surface. Nothing here decides anything; the substance of the node
is in [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md). A rebuild has no reason to keep
this as a separate unit.

## State

`Stateless.` — every entry reads a field of the upgrade node.

## Accessors of `Upgrade`

**Contract** — `section` is the configuration section whose keys are merged into the item
when this upgrade installs; `parent_group` and `parent_group_id` name the group the node
hangs under, which is how the tree is walked upward; `icon_name`, `name` and
`description_text` are the presentation triple the upgrade screen draws;
`get_highlight` reports whether the node is currently lit as part of a highlighted path;
`get_scheme_index` is the node's cell in the item's upgrade scheme grid, which is how the
screen maps a clicked cell back to an upgrade.

**Invariant** — `get_property_name(index)` is only defined for an index below the node's
fixed property count; the caller is trusted, not checked, in a shipping build.
