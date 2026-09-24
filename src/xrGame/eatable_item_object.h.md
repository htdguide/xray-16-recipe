# src/xrGame/eatable_item_object.h

> Declares the concrete consumable world object implemented in [`eatable_item_object.cpp`](eatable_item_object.cpp.md).

**Needs** — [`eatable_item.h`](eatable_item.h.md) · [`physic_item.h`](physic_item.h.md)
**Used by** — [`FoodItem.cpp`](FoodItem.cpp.md) · [`FoodItem.h`](FoodItem.h.md) · [`antirad.cpp`](antirad.cpp.md) · [`antirad.h`](antirad.h.md) · [`eatable_item_object.cpp`](eatable_item_object.cpp.md) · [`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md) · [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`medkit.cpp`](medkit.cpp.md) · [`medkit.h`](medkit.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CEatableItemObject`, the spawnable entity class for food, drink and medical items:
the [consumable behaviour](eatable_item.h.md) joined to the
[physical inventory item](physic_item.h.md). Substance — such as it is — in
[`eatable_item_object.cpp`](eatable_item_object.cpp.md).

The declaration is long and the implementation is almost entirely forwarding, because joining
two behaviour bases means every lifecycle point has to be given an explicit order. The
*orders* are the content; see the implementation twin.

Exported units, grouped by what they are for:

- the **cast family** — `cast_physics_shell_holder`, `cast_inventory_item`,
  `cast_attachable_item`, `cast_game_object`, and the four that answer *no*
  (`cast_weapon`, `cast_food_item`, `cast_missile`, `cast_hud_item`, `cast_weapon_ammo`).
  This is how the engine asks what an object is; a consumable declares itself an inventory
  item and an attachable physical object and denies being a weapon or a heads-up item.
  Notably it denies being a *food item*, which is a separate, older class.
- the **entity lifecycle** — `net_Spawn`, `net_Destroy`, `reinit`, `reload`, `Load`, `save`,
  `load`, `net_Import`, `net_Export`, `net_SaveRelevant`.
- the **frame and event hooks** — `UpdateCL`, `OnEvent`, `Hit`, `renderable_Render`.
- the **container transitions** — `OnH_B_Chield` / `OnH_A_Chield` (entering a container) and
  `OnH_B_Independent` / `OnH_A_Independent` (leaving one).
- the **physics bracket** — `activate_physic_shell`, `on_activate_physic_shell`, and the
  four correction-prediction hooks that bracket the multiplayer physics reconciliation,
  plus `make_Interpolation`.
- `Useful`, `NeedToDestroyObject`, `ef_weapon_type` — is it worth keeping, should it be
  destroyed, and its weapon-evaluation category (none).
- `use_parent_ai_locations` — while attached, the item's navigation position is its
  carrier's.
