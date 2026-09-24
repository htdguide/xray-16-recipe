# src/xrGame/inventory_upgrade_group.h

> Declares a group — a set of mutually exclusive upgrades gated behind a set of parent upgrades — implemented in [`inventory_upgrade_group.cpp`](inventory_upgrade_group.cpp.md).

**Needs** — [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`inventory_upgrade_group_inline.h`](inventory_upgrade_group_inline.h.md)
**Used by** — [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md) · [`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md) · [`inventory_upgrade_group.cpp`](inventory_upgrade_group.cpp.md) · [`inventory_upgrade_group_inline.h`](inventory_upgrade_group_inline.h.md) · [`inventory_upgrade_inline.h`](inventory_upgrade_inline.h.md) · [`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) · [`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

The middle layer of the upgrade forest. A group names a set of **included upgrades** that are
mutually exclusive — install one and the others become permanently unavailable — and carries
a set of **parent upgrades**, each of which unlocks it. Substance is in
[`inventory_upgrade_group.cpp`](inventory_upgrade_group.cpp.md).

A group is not a node the player sees. It exists so that the two rules "you need this
first" and "you may pick only one of these" are expressed once, in data, rather than as a
precondition script on every upgrade.

Exported units:

- `Group` — the set.
- `construct` — read the authored element list and ask the manager for each upgrade.
- `add_parent_upgrade` — attach another node that unlocks this group; duplicates collapse.
- `id` / `id_str` — the group's configuration section name.
- `fill_root` — pour every included upgrade into an owning root's flat lookup.
- `can_install` — the two structural rules, evaluated for one candidate upgrade.
- `highlight_up` / `highlight_down` — the forward and backward halves of the screen's
  dependency highlight.
