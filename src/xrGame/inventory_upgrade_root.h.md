# src/xrGame/inventory_upgrade_root.h

> Declares the per-item root of an upgrade tree: the node that owns the item's layout scheme and the flat list of every upgrade reachable from it.

**Needs** — [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) · [`inventory_upgrade_root_inline.h`](inventory_upgrade_root_inline.h.md)
**Used by** — [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md) · [`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) · [`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md) · [`inventory_upgrade_root_inline.h`](inventory_upgrade_root_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `inventory::upgrade::Root`, a tree node specialized as the entry point for one
item section. See [`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md).

Exported units:

- `Root` — extends the shared node with the item's layout scheme name and a flat list of
  every upgrade contained anywhere beneath it.
- `construct` — reads the item's `upgrades` and `upgrade_scheme` keys and builds the tree.
- `scheme` — the layout scheme name, in [`inventory_upgrade_root_inline.h`](inventory_upgrade_root_inline.h.md).
- `add_upgrade` — registers a descendant into the flat list.
- `is_root` — answers true, where the shared node answers false.
- `contain_upgrade` — whether a named upgrade is anywhere in this tree.
- `verify_scheme_index` · `get_upgrade_by_index` — the scheme grid's collision check and
  its reverse lookup.
- `highlight_hierarchy` · `reset_highlight` — the upgrade screen's path lighting.
- `log_hierarchy` · `test_all_upgrades` — diagnostics, development builds only.
