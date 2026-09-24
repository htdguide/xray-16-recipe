# src/xrGame/inventory_upgrade_manager.h

> Declares the registry that owns every upgrade tree in the game and the operations that install upgrades onto items.

**Needs** — [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`inventory_upgrade_manager_inline.h`](inventory_upgrade_manager_inline.h.md)
**Used by** — [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md) · [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md) · [`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md) · [`inventory_upgrade_group.cpp`](inventory_upgrade_group.cpp.md) · [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) · [`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) · [`inventory_upgrade_property.cpp`](inventory_upgrade_property.cpp.md) · [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md) · [`UIInvUpgrade.cpp`](ui/UIInvUpgrade.cpp.md) · [`UIInvUpgradeProperty.cpp`](ui/UIInvUpgradeProperty.cpp.md) · [`UIInventoryUpgradeWnd.cpp`](ui/UIInventoryUpgradeWnd.cpp.md) · [`UIInventoryUpgradeWnd.h`](ui/UIInventoryUpgradeWnd.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `inventory::upgrade::Manager`. See
[`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) for the substance: the
manager is a process-wide singleton-by-convention that reads every upgradeable item's tree
out of configuration at construction and then answers queries and performs installs
against it.

Exported units:

- `Manager` — owns four by-name tables: roots (one per upgradeable item section), groups,
  upgrades and properties. Every node is owned here and freed here; the tree links between
  nodes are non-owning.
- `get_root` · `get_group` · `get_upgrade` · `get_property` — by-name lookup, absent is not
  an error.
- `add_root` · `add_group` · `add_upgrade` · `add_property` — construct a node into its
  table and then let it construct itself, which is what lets a node reach back into the
  manager to create its own children.
- `make_known_upgrade` · `is_known_upgrade` — the per-upgrade "the player has seen this"
  flag, in an item-scoped and a bare form.
- `can_install_upgrade` · `can_add_upgrade` · `upgrade_install` · `upgrade_add` — the
  install path and its dry run.
- `init_install` — applies an item's authored default upgrades at spawn.
- `compute_range` — the low and high value of one tuned parameter across every item and
  every upgrade, for normalizing a bar in the upgrade screen.
- `get_item_scheme` · `get_upgrade_by_index` — the upgrade screen's layout lookups.
- `highlight_hierarchy` · `reset_highlight` — the screen's path highlighting.
- `log_hierarchy` · `test_all_upgrades` — diagnostics, present only in development builds.

**Notes** — the property table is public while the other three are private. That is an
accident of the upgrade screen needing to iterate properties directly; a rebuild should
expose an iteration accessor instead.
