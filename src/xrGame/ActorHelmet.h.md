# src/xrGame/ActorHelmet.h

> Declares the helmet implemented in [`ActorHelmet.cpp`](ActorHelmet.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`BoneProtections.h`](BoneProtections.h.md)
**Used by** — [`ActorHelmet.cpp`](ActorHelmet.cpp.md) · [`CustomOutfit.cpp`](CustomOutfit.cpp.md) · [`CustomOutfit_script.cpp`](CustomOutfit_script.cpp.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`Torch.cpp`](Torch.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`UIHudStatesWnd.cpp`](ui/UIHudStatesWnd.cpp.md) · [`UIInventoryUpgradeWnd.cpp`](ui/UIInventoryUpgradeWnd.cpp.md) · [`UIItemInfo.cpp`](ui/UIItemInfo.cpp.md) · [`UIMainIngameWnd.cpp`](ui/UIMainIngameWnd.cpp.md) · [`UIOutfitInfo.cpp`](ui/UIOutfitInfo.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CHelmet`, the worn item that reduces damage to the head and gates night vision.
Substance is in [`ActorHelmet.cpp`](ActorHelmet.cpp.md).

Exported units:

- `CHelmet` — the item. Holds a protection factor per damage type, a per-bone protection
  table, five restore-rate bonuses, a stamina-loss factor, and the names of the night-vision
  and per-bone configuration sections.
- `Load` / `install_upgrade_impl` — read tuning, and apply an upgrade over it.
- `HitThroughArmor` — the damage-reduction formula, in three generation-specific variants.
- `GetHitTypeProtection` / `GetDefHitTypeProtection` / `GetBoneArmor` — the protection
  readouts, condition-scaled.
- `Hit` — wears the item.
- `ReloadBonesProtection` / `AddBonesProtection` — build or extend the per-bone table
  against a skeleton.
- `net_Spawn` / `net_Export` / `net_Import` — spawn-time table construction and the
  one-byte condition on the wire.
- `OnMoveToSlot` / `OnMoveToRuck` — night-vision on and off.
- `OnH_A_Chield` — the became-a-child hook; it delegates and does nothing else.

## Notes

The protection list is declared mutable and queried through const methods, which is a
signal that the per-bone table is built lazily. A rebuild should build it at spawn and make
the queries pure.
