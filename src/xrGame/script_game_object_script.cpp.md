# src/xrGame/script_game_object_script.cpp

> Assembles the game object's script registration from the chain of partial registrations, and declares the two constant namespaces that go with it: the sight parameters and the callback event names.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md) · [`script_game_object_script_trader.cpp`](script_game_object_script_trader.cpp.md) · [`game_object_space.h`](game_object_space.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`script_ini_file.h`](../xrServerEntities/script_ini_file.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

The head of the registration chain. The [game object facade](script_game_object.h.md) has
several hundred exported methods and the declaration could not be built in one translation
unit, so it is threaded through four functions, each adding its share and returning the
declaration. This file creates the declaration, runs it through the chain, and adds the
things that belong to no particular slice: the sight parameter record and the **callback
event table**, which is the most consequential list in the file.

## `script_register`

**Contract** — creates a class declaration named `game_object`, threads it through the
chain — trader slice, then the first slice, then the second — and registers alongside it:

```text
class CSightParams              # a read-only snapshot of what a creature is looking at
  m_object, m_vector, m_sight_type
  CSightParams.<sight kinds> = {
    current_direction, path_direction, direction, position, object,
    cover, search, look_over, cover_look_over,
    fire_object, fire_position, animation_direction, dummy }

class callback                  # no fields: a namespace for the event names
  callback.callback_types = { ...see below... }

free functions: buy_condition, sell_condition (two shapes each), show_condition
```

**Invariants**

- The chain order is fixed and the nesting is what fixes it. Order does not affect the
  resulting surface, only which unit compiles what; a rebuild registers the whole surface
  once and has no chain.
- The sight parameter fields are **read-only** to script. The record is a snapshot of a
  creature's current sight action; writing to it would change nothing, so it is not
  offered — a script changes sight through the facade's own sight setters.
- The free trade-condition functions set the **engine-wide defaults**, where the facade's
  same-named methods set one character's. Exporting both under the same names, one as a
  method and one as a bare function, is frozen and scripts rely on the distinction.

## The callback event table

**Contract** — the names by which a script addresses an object's callback slots. They fall
into groups, and the grouping is the model:

```text
trade         : trade_start, trade_stop, trade_sell_buy_item, trade_perform_operation
trader        : trader_global_anim_request, trader_head_anim_request, trader_sound_end
zones/borders : zone_enter, zone_exit, level_border_enter, level_border_exit
life          : death, actor_before_death, actor_sleep
navigation    : patrol_path_in_point
pda / info    : inventory_pda, inventory_info, inventory_info_removed, article_info,
                task_state, map_location_added
interaction   : use_object, hit, sound
action queue  : action_removed, action_movement, action_watch, action_animation,
                action_sound, action_particle, action_object, script_animation
items         : on_item_take, on_item_drop, take_item_from_box,
                item_to_belt, item_to_slot, item_to_ruck
weapon        : weapon_no_ammo, weapon_jammed, weapon_zoom_in, weapon_zoom_out,
                weapon_magazine_empty, hud_animation_end
vehicles      : helicopter_on_point, helicopter_on_hit,
                on_attach_vehicle, on_detach_vehicle, on_use_vehicle
input         : key_press, key_release, key_hold, mouse_move, mouse_wheel,
                controller_press, controller_release, controller_hold
                (each also spelled with an "on_" prefix, same value)
```

**Invariants**

- The names are **frozen and the values are not**: scripts address slots by name and the
  underlying numbering is internal. A rebuild may renumber freely and must not rename.
- The trader animation requests are *requests*, not notifications: the engine asks the
  script which animation to play and uses the answer, so they are decision callbacks like
  the smart-cover target selector.
- **The action-queue events fire per channel.** A script driving an entity through the
  action queue learns which channel finished, not merely that the action did, which is what
  lets it react to a sound ending while movement continues.
- Five input events carry two spellings each, bare and prefixed, because two mod
  communities added them independently under different conventions and both shipped. Same
  value, both frozen.

**Notes**

The input, vehicle, weapon and inventory groups postdate the original games and were added
by the open-source engine's contributors — the original had no way for a script to see a
key press. That is why the naming conventions diverge within one table: the older names are
bare verbs and the newer ones carry prefixes. A rebuild keeps every spelling and should not
normalize them.

The constant tables hang off empty types here for the same reason as everywhere else in
this directory — the binding layer has no standalone namespace object. The sight kinds are
attached to the sight parameter record under a placeholder table name that no script uses,
since in this binding layer a class's constants are reachable directly from the class.
