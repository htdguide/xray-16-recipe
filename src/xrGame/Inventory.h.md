# src/xrGame/Inventory.h

> Declares the inventory, its slot record and the quick-switch priority group, implemented in [`Inventory.cpp`](Inventory.cpp.md) and [`inventory_quickswitch.cpp`](inventory_quickswitch.cpp.md).

**Needs** — [`inventory_item.h`](inventory_item.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorBackpack.cpp`](ActorBackpack.cpp.md) · [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorHelmet.cpp`](ActorHelmet.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Actor_Feel.cpp`](Actor_Feel.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`Artefact.cpp`](Artefact.cpp.md) · [`Car.cpp`](Car.cpp.md) · _and 80 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CInventory` — one entity's carried belongings — plus `CInventorySlot` (one typed
position) and `priority_group` (a named set of sections used by the multiplayer quick-switch
ordering). Substance is in [`Inventory.cpp`](Inventory.cpp.md); the quick-switch and grenade
cycling half is implemented in
[`inventory_quickswitch.cpp`](inventory_quickswitch.cpp.md).

The shape this header fixes, and the reason to read it before the implementation: the three
storage areas and the `all` list are **public fields**, not an encapsulated store. Every
screen, script binding and trade routine in the game reaches straight into them. That is the
single biggest constraint on a rebuild of this module — the containers and their iteration
order are effectively part of the public contract, so replacing them with a different
structure breaks callers far outside this file.

Exported units:

- `CInventorySlot` — an item, a persistent flag (worn rather than carried) and an
  activatable flag (may be the thing in the hands).
- `priority_group` — a named set of section names, with membership test; the quick-switch
  order for one slot is five of these.
- `CInventory` — the inventory.
- `Take` / `DropItem` / `Clear` — items entering and leaving.
- `Slot` / `Belt` / `Ruck` — the three placements, each also an eviction from wherever the
  item was.
- `CanPutInSlot` / `CanPutInBelt` / `CanPutInRuck` / `CanTakeItem` — the predicates.
- `InSlot` / `InBelt` / `InRuck` — where an item currently is.
- `Activate` / `ActiveItem` / `ItemFromSlot` / `GetActiveSlot` / `GetNextActiveSlot` /
  `GetPrevActiveSlot` / `SetActiveSlot` / `SetPrevActiveSlot` — the in-the-hands state
  machine. Three slot values, not one, because a switch takes an animation to complete.
- `FirstSlot` / `LastSlot` / `SlotIsPersistent` — the slot range, inclusive at both ends,
  with index 0 reserved as the "no slot" sentinel.
- `Update` — per-frame: completes a pending hand switch and runs the drop poll.
- `Action` / `ActiveWeapon` — the input funnel.
- `Eat` / `ClientEat` — consuming an item, authoritative and client halves.
- `Same` / `SameSlot` / `Get` (by name, by identifier, by class) / `GetAny` / `item` /
  `get_object_by_id` / `GetItemFromInventory` — the lookup surface.
- `dwfGetSameItemCount` / `dwfGetGrenadeCount` / `bfCheckForObject` / `dwfGetObjectCount` /
  `tpfGetObjectByIndex` — the counting and indexing surface, used from script.
- `TotalWeight` / `CalcTotalWeight` / `GetMaxWeight` / `SetMaxWeight` — the weight cache.
- `BeltWidth` / `BeltMaxWidth` — belt capacity, which in the newer games is a property of
  the worn outfit.
- `IsSlotsUseful` / `SetSlotsUseful` / `IsBeltUseful` / `SetBeltUseful` — whether this owner
  has slots and a belt at all.
- `SetSlotsBlocked` / `BlockSlot` / `UnblockSlot` / `IsSlotBlocked` — the per-slot block
  counters. Counters rather than flags so overlapping reasons nest.
- `AddAvailableItems` — what a trade or corpse-search screen may show.
- `ModifyFrame` / `InvalidateState` — the change stamp the inventory screen polls.
- `Items_SetCurrentEntityHud` — rebuild weapon attachment visuals on a viewpoint change.
- `isBeautifulForActiveSlot` — does anything held declare this item necessary.
- `m_available_grenade_types` / `HasNextGrenade` / `GetNextGrenade` / `ActivateNextGrenade` /
  `ActivateNextGrenadeDeffered` — grenade cycling by section rather than by slot.
- `GetNextItemInActiveSlot` / `ActivateNextItemInActiveSlot` / `GetPriorityGroup` /
  `InitPriorityGroupsForQSwitch` — the multiplayer quick-switch ordering, five priority
  levels deep for each of two weapon slots.
- `m_all` / `m_ruck` / `m_belt` / `m_activ_last_items` — the containers, public.

## Notes

`m_activ_last_items` is declared and never written anywhere in the tree. Its name suggests a
recently-held history for a "previous weapon" key that was never finished.
