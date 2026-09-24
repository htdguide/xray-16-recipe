# src/xrGame/inventory_upgrade_property.h

> Declares the descriptor of one upgrade *property* — a named, icon-bearing category of item parameter whose displayed value is computed by a script.

**Needs** — [`inventory_upgrade.h`](inventory_upgrade.h.md) · [`inventory_upgrade_property_inline.h`](inventory_upgrade_property_inline.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`inventory_upgrade.cpp`](inventory_upgrade.cpp.md) · [`inventory_upgrade_manager.cpp`](inventory_upgrade_manager.cpp.md) · [`inventory_upgrade_property.cpp`](inventory_upgrade_property.cpp.md) · [`inventory_upgrade_property_inline.h`](inventory_upgrade_property_inline.h.md) · [`UIInvUpgradeProperty.cpp`](ui/UIInvUpgradeProperty.cpp.md) · [`UIInvUpgradeProperty.h`](ui/UIInvUpgradeProperty.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `inventory::upgrade::Property`. See
[`inventory_upgrade_property.cpp`](inventory_upgrade_property.cpp.md) for the substance.

Exported units:

- `Property` — identity, display name, icon and tint, a bound script function, and the
  list of item parameter names the property covers.
- `construct` — reads the descriptor out of its configuration section and binds the script
  function.
- `run_functor` — formats one parameter's value through that script function.
- the accessors, in [`inventory_upgrade_property_inline.h`](inventory_upgrade_property_inline.h.md).
