# src/xrGame/player_hud.h

> Declares the first-person hands rig and everything attached to it — implemented in [`player_hud.cpp`](player_hud.cpp.md).

**Needs** — [`firedeps.h`](firedeps.h.md) · [`actor_defs.h`](actor_defs.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`AdvancedDetector.cpp`](AdvancedDetector.cpp.md) · [`CustomDetector.cpp`](CustomDetector.cpp.md) · [`CustomOutfit.cpp`](CustomOutfit.cpp.md) · [`EliteDetector.cpp`](EliteDetector.cpp.md) · [`HUDManager.cpp`](HUDManager.cpp.md) · [`HudItem.cpp`](HudItem.cpp.md) · [`Inventory.cpp`](Inventory.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`SimpleDetector.cpp`](SimpleDetector.cpp.md) · [`Weapon.cpp`](Weapon.cpp.md) · _and 8 more_
**Tier floor** — T2: a declaration plus the authored-measurement record

## Purpose

Declares the player's first-person view rig: one arms model, up to two attached items, and
the measurements that place each item in the arms' hands. Substance is in
[`player_hud.cpp`](player_hud.cpp.md).

Three shapes are fixed here and are worth reading before the implementation.

**There are exactly two attachment places.** Index 0 is the main hand, index 1 the off hand.
Not a list — a pair — and the index is a property of the item's configuration, so an item
knows which hand it goes in. Everything in the implementation that looks like duplication is
this pair being handled twice.

**A held item is either monolithic or two-part.** A *two-part* item has a separate arms model
that the rig animates and an item model attached to a bone of it; a *monolithic* item's model
contains the arms, so the rig's own arms are hidden and the item's model is animated instead.
Which one an item is is decided by which visual key its configuration carries, and nearly
every branch in the implementation is this distinction.

**A motion is an alias, not a name.** Configuration names a logical animation
(`anm_reload`), which resolves to a base clip name, an optional second clip name for the
item's own model, a speed, and up to nine numbered variants that are chosen at random. That
indirection is what lets one item have three different reload animations without the game
logic knowing.

## State

The authored measurements are the substantial record here:

```text
RECORD HudItemMeasures
  hands_offset : list of 2 lists of 3 vectors   # [position|rotation][hip, aim, launcher]
  hands_attach : (position, rotation)           # where the arms sit relative to the camera
  item_attach  : (position, rotation)           # where the item sits relative to its anchor
  fire_point_offset, fire_point2_offset, shell_point_offset : vector
  fire_bone, fire_bone2, shell_bone : int       # bones those offsets are relative to
  prop_flags   : set of {has_fire_point, has_fire_point2, has_shell_point, widescreen_now}
  inertion     : InertionParams

RECORD InertionParams          # eight numbers governing how the weapon lags the camera
  pitch_offset_r, pitch_offset_n, pitch_offset_d : real   # sideways, vertical, depth
  pitch_low_limit  : real
  origin_offset, origin_offset_aim : real      # how far the weapon swings
  tendto_speed, tendto_speed_aim   : real      # how fast it catches up
```

**Invariants**
- Each of the three named points exists exactly when its bone does. The pairing is asserted
  at load, because a point with no bone has nothing to be relative to.
- The three hand-offset slots are indexed by the item's current aiming mode — hip, aim,
  grenade launcher — and slot 0 is always zero, meaning "no offset at the hip".
- The widescreen flag records which aspect ratio the measurements were read for. They are
  re-read when it changes.

Exported units:

- `motion_descr` / `player_hud_motion` / `player_hud_motion_container` — the motion alias
  table: a logical name resolving to variants, a second name for the item's model, and a
  speed.
- `hud_item_measures` — the record above, plus `load` (two-part), `load_monolithic`,
  `load_inertion_params` and `update` (rebuild the attachment transform).
- `attachable_hud_item` — one item in one hand: its model, its measurements, its motion
  table, its runtime transform.
- `update` / `render` / `setup_firedeps` / `anim_play` / `reload_measures` /
  `set_bone_visible` / `tune` — what an attached item does.
- `hands_attach_pos` / `hands_attach_rot` / `hands_offset_pos` / `hands_offset_rot` — the
  bind pose and the current aiming offset.
- `player_hud` — the rig.
- `load` / `load_default` — bring up an arms model by configuration section.
- `attach_item` / `detach_item` / `detach_item_idx` / `detach_all_items` /
  `attached_item` / `allow_activation` — the two attachment places.
- `update` / `render_hud` / `render_item_ui` / `render_item_ui_query` — the per-frame work.
- `anim_play` / `motion_length` — playing a motion and asking how long one lasts.
- `calc_transform` — place an item at its anchor bone.
- `OnMovementChanged` — tell held items the player started or stopped moving.
- `create_hud_item` — the cache of constructed items, keyed by configuration section.
- `g_player_hud` — the single process-wide rig.

## Notes

`m_ancors` — the bones of the arms model that items attach to, one per attachment place,
collected from configuration keys beginning `ancor_`. The misspelling is in the shipped
configuration and therefore frozen.
