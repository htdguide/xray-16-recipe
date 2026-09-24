# src/xrGame/inventory_upgrade_base.h

> Declares what every node of an item's upgrade forest has in common — an identifier, a known/unknown flag, and the dependent groups it unlocks — plus the frozen verdict enumeration the whole mechanic answers in.

**Needs** — [`inventory_upgrade_base_inline.h`](inventory_upgrade_base_inline.h.md) · [`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md)
**Used by** — [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md) · [`inventory_upgrade.h`](inventory_upgrade.h.md) · [`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md) · [`inventory_upgrade_base_inline.h`](inventory_upgrade_base_inline.h.md) · [`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md) · [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md)
**Tier floor** — T3: a declaration and one enumeration

## Purpose

The upgrade forest has two kinds of node that share a base: a **root** (one per upgradeable
item section) and an **upgrade** (one purchasable modification). Both own a list of
*dependent groups* — the groups they unlock — and both answer `can_install`. This header is
where that commonality is named; the behaviour is in
[`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md).

One thing here is substance rather than declaration, and it is the reason to read this file
before the rest of the mechanic.

## State

```text
ENUM UpgradeStateResult          # why an upgrade may not be installed
  result_ok
  result_e_unknown               # the player has not learned this upgrade exists
  result_e_installed             # already on the item
  result_e_parents               # a prerequisite upgrade or group is missing
  result_e_group                 # a sibling in the same group is already installed
  result_e_precondition_money    # the script precondition refused on cost grounds
  result_e_precondition_quest    # the script precondition refused on story grounds
  result_e_cant_do               # the script precondition refused generically
```

The order is not arbitrary: the upgrade screen maps each value to its own refusal message
and its own colour, and the two `precondition` values are what the script layer's numeric
refusal codes are translated into — a translation that differs between games, and is done
in [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md).

```text
RECORD UpgradeBase
  id              : text        # the configuration section naming this node
  known           : bool        # has the player learned of this upgrade
  depended_groups : list<Group> # the groups this node unlocks
```

Exported units:

- `UpgradeStateResult` — the verdict enumeration above.
- `construct` — bind the node to its identifier.
- `id` / `id_str` / `is_known` / `make_known` — identity and the learned flag.
- `is_root` — whether this node is the per-item root rather than a purchasable upgrade.
- `contain_upgrade` — identity test by upgrade name.
- `fill_root_container` — the recursive walk that collects every node reachable from a root.
- `can_install` — the two checks every node shares.
- `add_dependent_groups` — parse a group-name list and attach the named groups.
- `highlight_up` / `highlight_down` — the two directions of the screen's dependency
  highlight; no-ops at this level.
