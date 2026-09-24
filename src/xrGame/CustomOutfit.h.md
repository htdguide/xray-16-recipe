# src/xrGame/CustomOutfit.h

> Declares the body armour implemented in [`CustomOutfit.cpp`](CustomOutfit.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`BoneProtections.h`](BoneProtections.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`CustomOutfit.cpp`](CustomOutfit.cpp.md) · [`CustomOutfit_script.cpp`](CustomOutfit_script.cpp.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`ExoOutfit.cpp`](ExoOutfit.cpp.md) · [`ExoOutfit.h`](ExoOutfit.h.md) · [`Inventory.cpp`](Inventory.cpp.md) · [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`MilitaryOutfit.cpp`](MilitaryOutfit.cpp.md) · [`MilitaryOutfit.h`](MilitaryOutfit.h.md) · [`ScientificOutfit.cpp`](ScientificOutfit.cpp.md) · [`ScientificOutfit.h`](ScientificOutfit.h.md) · [`StalkerOutfit.h`](StalkerOutfit.h.md) · _and 11 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares the outfit: a per-damage-type protection table, a per-bone armour table bound to
the wearer's skeleton, and the modifiers an outfit applies to its wearer. Substance in
[`CustomOutfit.cpp`](CustomOutfit.cpp.md).

Exported units:

- `CCustomOutfit` — the armour item.
- `Load`, `net_Spawn`, `net_Export`, `net_Import`, `install_upgrade_impl` — configuration,
  the condition as the only replicated field, and the workbench upgrade path.
- `HitThroughArmor` — **the damage formula**: three of them, selected per outfit by
  inferring which game's data this is.
- `Hit` — wear the outfit down by absorbed damage.
- `GetHitTypeProtection`, `GetDefHitTypeProtection`, `GetBoneArmor`, `GetHitFracType` —
  the protection queries, per type, per type-and-bone, per bone, and which formula.
- `BonePassBullet` — this bone is unarmoured; bullets pass straight through.
- `ReloadBonesProtection`, `AddBonesProtection` — bind or merge a bone table against the
  wearer's skeleton.
- `GetPowerLoss` — the stamina drain multiplier, which a ruined outfit cancels under the
  two older damage formulas.
- `OnMoveToSlot`, `OnMoveToRuck`, `OnH_A_Chield` — dressing and undressing: the model
  swap, the first-person arms, the helmet eviction and the night-vision handover.
- `ApplySkinModel` — the model and arms swap, with the multiplayer per-team skin override.
- `ef_equipment_type`, `GetFullIconName`, `get_artefact_count` — the classification, the
  large inventory image, and how many artefact containers this outfit provides.
- Script registration: the restore rates, the stamina penalty and the carry bonuses as
  *writable* fields; see [`CustomOutfit_script.cpp`](CustomOutfit_script.cpp.md).
