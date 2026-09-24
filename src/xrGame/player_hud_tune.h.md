# src/xrGame/player_hud_tune.h

> Declares the in-game tool that lets an artist position a weapon in the player's hands live and paste the result back into configuration. Implemented in [`player_hud_tune.cpp`](player_hud_tune.cpp.md).

**Needs** — [`player_hud.h`](player_hud.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`GamePersistent.h`](GamePersistent.h.md) · [`player_hud_tune.cpp`](player_hud_tune.cpp.md)
**Tier floor** — T3: a declaration and two label tables

## Purpose

Declares `CHudTuner`, one tool in the engine's debug overlay. It edits the eleven authored
measurements that place a held item — see
[`player_hud.h`](player_hud.h.md) — against the live view, and produces the configuration
text for what the artist arrived at.

The label tables here are not decoration: they name the eleven editable quantities in the
same order the tool presents them, and the names are the closest thing in the tree to
documentation of what each authored key does.

```text
ENUM TunableMeasurement
  hud_position       # where the arms sit, resting
  hud_rotation
  hud_position_aim   # the offset applied while aiming
  hud_rotation_aim
  hud_position_gl    # the offset applied while the grenade launcher is selected
  hud_rotation_gl
  item_position      # where the item sits on its anchor bone
  item_rotation
  fire_point         # muzzle
  fire_point_2       # second muzzle
  shell_point        # ejection port
```

## State

```text
RECORD HudTuner
  current_item     : optional<attachable_hud_item>   # whichever hand is being edited
  current_hand     : {main, off}
  saved_measures   : HudItemMeasures    # what the item had when editing started
  edited_measures  : HudItemMeasures    # what the sliders hold now
  position_step    : real               # 0.0005 by default
  rotation_step    : real               # 0.05 by default
  point_size       : real               # 0.005 by default
  paused           : bool
  draw_fire_point, draw_fire_point_2, draw_fire_direction,
  draw_fire_direction_2, draw_shell_point : bool
```

Exported units:

- `CHudTuner` — the tool.
- `on_tool_frame` — draw the panel and apply whatever it edited, once per frame.
- `is_active` — whether the tuner is currently editing, which the rig polls.
- `tool_name` — its name in the overlay's tool list.
- `ResetToDefaultValues` / `UpdateValues` — reload from configuration, and push the edits
  into the live item.
