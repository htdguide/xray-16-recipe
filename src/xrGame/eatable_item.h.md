# src/xrGame/eatable_item.h

> Declares the consumable-item mixin implemented in [`eatable_item.cpp`](eatable_item.cpp.md).

**Needs** — [`inventory_item.h`](inventory_item.h.md)
**Used by** — [`Inventory.cpp`](Inventory.cpp.md) · [`actor_mp_client.cpp`](actor_mp_client.cpp.md) · [`ai_rat.h`](ai/monsters/rats/ai_rat.h.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`eatable_item.cpp`](eatable_item.cpp.md) · [`eatable_item_object.cpp`](eatable_item_object.cpp.md) · [`eatable_item_object.h`](eatable_item_object.h.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md) · [`UIItemInfo.cpp`](ui/UIItemInfo.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CEatableItem`, the behaviour shared by everything the player or a creature can
consume: food, drink, medical kits, bandages, anti-radiation drugs. Substance in
[`eatable_item.cpp`](eatable_item.cpp.md).

Exported units:

- `Load` — the four tunables: number of uses, whether the item disappears when spent, and the
  full and empty weights.
- `UseBy` — consume one use, applying the item's condition influences and any boosters.
- `Useful` / `Empty` / `CanDelete` — whether the item is worth keeping, whether it is spent,
  and whether being spent means deletion.
- `GetMaxUses` / `GetRemainingUses` / `SetRemainingUses` — the use counter, with the setter
  clamped to the maximum.
- `Weight` — the item's weight, interpolated between full and empty by remaining uses.
- `save` / `load` — the remaining-use count is the only persisted field.
- `net_Spawn`, `OnH_B_Independent`, `OnH_A_Independent` — the lifecycle points at which a
  spent item removes itself from the world.
- `cast_eatable_item` — the downcast that identifies an item as consumable. The engine's
  type discrimination is by a family of such casts rather than by a type tag.

**Notes** — the mixin keeps a pointer back to the physical item it is combined into, because
the configuration section and the world object belong to that half. See
[`eatable_item_object.cpp`](eatable_item_object.cpp.md) for how the two halves are joined.
