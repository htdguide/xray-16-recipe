# src/xrGame/inventory_upgrade.h

> Declares one purchasable upgrade — the leaf of the forest — implemented in [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md).

**Needs** — [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) · [`inventory_upgrade_inline.h`](inventory_upgrade_inline.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md) · [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md) · [`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md) · [`inventory_upgrade_group.cpp`](inventory_upgrade_group.cpp.md) · [`inventory_upgrade_inline.h`](inventory_upgrade_inline.h.md) · [`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) · [`inventory_upgrade_property.h`](inventory_upgrade_property.h.md) · [`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md) · [`UIInvUpgrade.cpp`](ui/UIInvUpgrade.cpp.md) · [`UIInvUpgrade.h`](ui/UIInvUpgrade.h.md) · [`UIInvUpgradeInfo.cpp`](ui/UIInvUpgradeInfo.cpp.md) · [`UIInvUpgradeInfo.h`](ui/UIInvUpgradeInfo.h.md) · [`UIInventoryUpgradeWnd.cpp`](ui/UIInventoryUpgradeWnd.cpp.md)
**Tier floor** — T3: a declaration plus three bound-script-call shapes

## Purpose

One node of an item's upgrade tree: a name, a description, an icon, a position in the
screen's grid, up to four *properties* (the named effects shown to the player), and three
script hooks — a precondition, an effect and a prerequisites-text producer. Substance is in
[`inventory_upgrade.cpp`](inventory_upgrade.cpp.md).

Two shapes are fixed here.

**At most four properties per upgrade.** The screen has room for four lines of effect text
and the data was authored against that; a fifth named property in configuration is silently
dropped.

**A script hook is a callable plus its frozen arguments.** Each of the three hooks is stored
as a bound script function together with the argument values it will always be called with —
the authored parameter string, the item's configuration section, and for the effect hook a
loading flag. They differ only in arity and return type. In C++ that produced a small tower
of templates inheriting from each other to add one argument at a time; the decision behind
it is simply that **each hook's arguments are decided once at construction and never vary at
the call site**, so the call is a nullary invocation from the caller's point of view. A
rebuild binds a closure and is done.

Exported units:

- `max_properties_count` — four.
- `Upgrade` — the node.
- `construct` — read every authored key and resolve the three script hooks.
- `section` / `parent_group_id` / `parent_group` / `icon_name` / `name` /
  `description_text` — the authored presentation and the section whose keys this upgrade
  merges into the item.
- `get_property_name` / `get_scheme_index` / `check_scheme_index` — the four property names
  and the node's cell in the upgrade screen's two-dimensional scheme.
- `get_prerequisites` — the script-produced text explaining what is still missing.
- `can_install` — the full verdict, combining the base checks, the group's checks and the
  script precondition.
- `can_add` — the base checks only, used where the group rules must not apply.
- `run_effects` — invoke the script effect hook, distinguishing a fresh install from a
  replay during load.
- `get_highlight` / `set_highlight` / `highlight_up` / `highlight_down` — the screen's
  dependency highlight.
- `fill_root_container` — register into the owning root's flat lookup, then recurse.

## Notes

A fourth hook — a tooltip producer — is declared, read from configuration and invoked in
commented-out code throughout this pair of files. The authored data carries its key. It was
disabled rather than removed, and a rebuild can ignore it.
