# src/xrServerEntities/xrServer_Objects_ALife_Items.h

> Declares the inventory-item mixin and every carryable record: weapons, ammunition, outfits, artefacts, documents, grenades.

**Needs** — [`xrServer_Objects_ALife.h`](xrServer_Objects_ALife.h.md) · [`PHSynchronize.h`](PHSynchronize.h.md) · [`inventory_space.h`](inventory_space.h.md) · [`character_info_defs.h`](character_info_defs.h.md) · [`InfoPortionDefs.h`](InfoPortionDefs.h.md)
**Used by** — [`Car.cpp`](../xrGame/Car.cpp.md) · [`InfoDocument.cpp`](../xrGame/InfoDocument.cpp.md) · [`PDA.cpp`](../xrGame/PDA.cpp.md) · [`RocketLauncher.cpp`](../xrGame/RocketLauncher.cpp.md) · [`Torch.cpp`](../xrGame/Torch.cpp.md) · [`Weapon.cpp`](../xrGame/Weapon.cpp.md) · [`WeaponAmmo.cpp`](../xrGame/WeaponAmmo.cpp.md) · [`alife_object.cpp`](../xrGame/alife_object.cpp.md) · [`alife_simulator_base2.cpp`](../xrGame/alife_simulator_base2.cpp.md) · [`game_cl_deathmatch_buywnd.cpp`](../xrGame/game_cl_deathmatch_buywnd.cpp.md) · [`game_sv_item_respawner.cpp`](../xrGame/game_sv_item_respawner.cpp.md) · [`inventory_item.h`](../xrGame/inventory_item.h.md) · [`xrServer_process_event.cpp`](../xrGame/xrServer_process_event.cpp.md) · [`xrServer_Objects_ALife_All.h`](xrServer_Objects_ALife_All.h.md) · _and 5 more_
**Tier floor** — T2: a hierarchy declaration; layouts are in the implementation.

## Purpose

Declares the second of the two concrete-record families. Layouts and contracts are in
[`xrServer_Objects_ALife_Items.cpp`](xrServer_Objects_ALife_Items.cpp.md).

The organizing idea is that **"is an item" is a mixin, not a level of the hierarchy**. A rat
is a monster *and* an inventory item (you can carry its corpse); a weapon is a visual dynamic
object *and* an inventory item. The mixin carries the economic and physical facts — mass,
cost, condition, upgrades, and the rigid-body state a dropped item needs on the wire — and
the record it is mixed into carries everything else.

## Exported units

- `CSE_ALifeInventoryItem` — the mixin: condition, mass, cost, nutrition, upgrades, and the
  dropped-item physics update.
- `CSE_ALifeItem` — a dynamic visual object that is an inventory item. The base of everything
  below.
- `CSE_ALifeItemTorch` — a torch with a night-vision mode.
- `CSE_ALifeItemAmmo` — a box of rounds.
- `CSE_ALifeItemWeapon` — magazine contents, attached addons, ammunition type, packed
  grenade-launcher load.
- `CSE_ALifeItemWeaponMagazined` — adds the selected fire mode.
- `CSE_ALifeItemWeaponMagazinedWGL` — adds the grenade-launcher toggle.
- `CSE_ALifeItemWeaponShotGun` — adds the per-round ammunition list in the tube.
- `CSE_ALifeItemWeaponAutoShotGun` — a naming distinction only.
- `CSE_ALifeItemDetector` — carries an evaluation-function detector type.
- `CSE_ALifeItemArtefact` — carries an anomaly value.
- `CSE_ALifeItemPDA` — original owner, specific character, info portion.
- `CSE_ALifeItemDocument` — one info portion.
- `CSE_ALifeItemGrenade`, `CSE_ALifeItemExplosive`, `CSE_ALifeItemBolt` — throwables.
- `CSE_ALifeItemCustomOutfit`, `CSE_ALifeItemHelmet` — worn protection.
- `EWeaponAddonState` — the three addon bits, which reach both data and script.
