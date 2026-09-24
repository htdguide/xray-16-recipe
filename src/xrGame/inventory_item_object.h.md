# src/xrGame/inventory_item_object.h

> Declares the concrete, spawnable "an item lying in the world" class — the join of the carryable mix-in and the physically simulated object — implemented in [`inventory_item_object.cpp`](inventory_item_object.cpp.md).

**Needs** — [`physic_item.h`](physic_item.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`inventory_item_object_inline.h`](inventory_item_object_inline.h.md)
**Used by** — [`ActorBackpack.cpp`](ActorBackpack.cpp.md) · [`ActorBackpack.h`](ActorBackpack.h.md) · [`ActorHelmet.cpp`](ActorHelmet.cpp.md) · [`ActorHelmet.h`](ActorHelmet.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`CustomOutfit.cpp`](CustomOutfit.cpp.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md) · [`ExplosiveItem.h`](ExplosiveItem.h.md) · [`GrenadeLauncher.cpp`](GrenadeLauncher.cpp.md) · [`GrenadeLauncher.h`](GrenadeLauncher.h.md) · [`InfoDocument.cpp`](InfoDocument.cpp.md) · [`InfoDocument.h`](InfoDocument.h.md) · [`MPPlayersBag.cpp`](MPPlayersBag.cpp.md) · _and 17 more_
**Tier floor** — T2: a declaration that fixes a diamond join

## Purpose

`CInventoryItem` is a mix-in: name, weight, cost, condition, grid footprint, upgrade list.
`CPhysicItem` is a world object: a visual, a rigid body, a place in the object registry.
Neither is spawnable alone. This class is the join, and it is the base of almost everything
the player can pick up — weapons, outfits, medkits, artefacts, quest tokens all descend from
it or from one of its descendants.

Substance is in [`inventory_item_object.cpp`](inventory_item_object.cpp.md), which is almost
entirely the ordering of that join.

The one thing fixed here rather than there is the **downcast table**: a set of questions any
game object answers about itself — *are you a weapon, a food item, a missile, a heads-up
display item, ammunition, an attachable, an inventory item, a physics holder*. This class
answers yes to inventory item, attachable, physics holder and game object, and no to the
rest; subclasses override the ones they become. The table is how the rest of the game asks
"what kind of thing is this" without a class-identifier switch, and it is a real design
decision: the set of questions is closed and adding a kind means adding a question to the
common base. A rebuild with a capability-query mechanism should keep the question set
closed the same way, because callers rely on the answers being cheap and total.

Exported units:

- `CInventoryItemObject` — the joined class.
- `_construct` — two-phase construction: run both halves' own construction before anything
  is loaded.
- `cast_inventory_item` / `cast_attachable_item` / `cast_physics_shell_holder` /
  `cast_game_object` — the yes answers.
- `cast_weapon` / `cast_food_item` / `cast_missile` / `cast_hud_item` / `cast_weapon_ammo` —
  the no answers, for subclasses to override.
- `Load` / `reload` / `reinit` — configuration load and the two re-entries.
- `Hit` — damage, routed to both halves.
- `OnH_B_Independent` / `OnH_A_Independent` / `OnH_B_Chield` / `OnH_A_Chield` — the four
  edges of changing owner, before and after in each direction.
- `UpdateCL` — the per-frame client update.
- `OnEvent` / `net_Spawn` / `net_Destroy` / `net_Import` / `net_Export` / `save` / `load` /
  `net_SaveRelevant` — the network and persistence surface.
- `renderable_Render` — draw.
- `activate_physic_shell` / `on_activate_physic_shell` — the item's physics body going live.
- `make_Interpolation` / `PH_B_CrPr` / `PH_I_CrPr` / `PH_A_CrPr` / `PH_Ch_CrPr` — the
  client-side prediction hooks around the physics step.
- `modify_holder_params` — how holding this item changes the holder's view range and field
  of view (binoculars, a scoped weapon).
- `NeedToDestroyObject` / `Useful` — lifetime and whether an artificial intelligence should
  want this.
- `ef_weapon_type` — the coarse weapon-class number the creature evaluators compare; zero,
  meaning "not a weapon".
- `use_parent_ai_locations` — when attached, the item's navigation position is its holder's.
