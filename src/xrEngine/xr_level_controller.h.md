# src/xrEngine/xr_level_controller.h

> Declares the action vocabulary every input consumer in the engine speaks, and the shape of a binding.

**Needs** — [`xr_level_controller.cpp`](xr_level_controller.cpp.md) · [`xr_input.h`](xr_input.h.md) · [`key_binding_registrator_script.cpp`](key_binding_registrator_script.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`GameFont.cpp`](GameFont.cpp.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`StringTable.cpp`](StringTable/StringTable.cpp.md) · [`editor_base.cpp`](editor_base.cpp.md) · [`editor_base_input.cpp`](editor_base_input.cpp.md) · [`editor_helper.h`](editor_helper.h.md) · [`key_binding_registrator_script.cpp`](key_binding_registrator_script.cpp.md) · [`xr_input.cpp`](xr_input.cpp.md) · [`xr_level_controller.cpp`](xr_level_controller.cpp.md) · [`ActorInput.cpp`](../xrGame/ActorInput.cpp.md) · [`Actor_Events.cpp`](../xrGame/Actor_Events.cpp.md) · [`CameraFirstEye.cpp`](../xrGame/CameraFirstEye.cpp.md) · [`CameraLook.cpp`](../xrGame/CameraLook.cpp.md) · _and 33 more_
**Tier floor** — T2: an enumeration and three small records.

## Purpose

Declares the surface implemented in
[`xr_level_controller.cpp`](xr_level_controller.cpp.md), and — the part that matters —
declares the **action enumeration itself**, which is the vocabulary every input consumer in
the engine, and every shipped script, speaks. It is included almost everywhere input is
handled.

## The action enumeration — frozen

The identifiers are exported to script by name (see
[`key_binding_registrator_script.cpp`](key_binding_registrator_script.cpp.md)) and their
*names* appear in `user.ltx`, so both are frozen. Their numeric values are not frozen
against the data — nothing on disk stores a numeric action — but they **are** frozen against
the parallel name table: the identifier is the index into it, asserted at startup.

The vocabulary, by group:

```text
MOVEMENT      look_around move_around (gamepad axes) · left right up down
              forward back lstrafe rstrafe · llookout rlookout
              jump crouch crouch_toggle accel sprint_toggle · turn_engine

CAMERA        cam_1 .. cam_4 · cam_zoom_in cam_zoom_out · cam_autoaim

EQUIPMENT     torch night_vision show_detector · wpn_1 .. wpn_6 · artefact
              wpn_next wpn_fire wpn_zoom wpn_zoom_inc wpn_zoom_dec
              wpn_reload wpn_func wpn_firemode_prev wpn_firemode_next
              next_slot prev_slot · use_bandage use_medkit · quick_use_1 .. _4

SESSION       pause drop use scores screenshot enter quit console
              inventory map contacts active_jobs ext_1 · quick_save quick_load
              alife_command · kick · editor

MULTIPLAYER   chat chat_team buy_menu skin_menu team_menu
              vote_begin vote vote_yes vote_no show_admin_menu
              speech_menu_0 .. _9

MODDING       custom1 .. custom15 · pda_tab1 .. pda_tab6

CONTEXTUAL    ui_*      : move (+ four directions, + a secondary axis), click_1/2,
                          accept, back, action_1/2, tab_prev/next, button_1..0
              pda_*     : map_move (+ directions), map_zoom_in/out/reset,
                          map_show_actor, map_show_legend, filter_toggle
              talk_*    : switch_to_trade, log_scroll (+ up/down)
```

**Notes** — the fifteen `custom` actions and six `pda_tab` actions exist **only** for
modifications: the engine binds and dispatches them and does nothing with them. They are a
deliberate extension point, and the count is the arbitrary part — fifteen because somebody
had to pick.

`look_around` and `move_around` are not keys but **gamepad axes** wearing the same type. An
action bound to an axis receives a magnitude rather than a press, which the input layer
handles; the binding layer does not distinguish them.

Two enumerators terminate the list and are not actions: one marks the count (and is the
size of every parallel table), and one is the "no action" answer a failed lookup returns.

## Other exported units

- **`Group`** — single-player only, multiplayer only, or both. Encoded as bits so that "both"
  is a subset of each, which is what makes the matching test cheap; the encoding is
  incidental, the three-way distinction is not.
- **`Context`** — none, interface, map, conversation. Selects which subset of actions a key
  event is dispatched against.
- **`Key`** — a frozen name, a scancode, and a display name the platform supplies.
- **`Binding`** — an action plus exactly three optional key slots: primary keyboard/mouse,
  secondary keyboard/mouse, gamepad. The slot count and its meaning by position are relied
  on by the settings file's three `bind` commands.
- **`GAME_ACTION_MARK`** — the escape byte (27) that introduces an action reference inside a
  piece of displayed text, so a caption can say "press ⟨use⟩" and have the binding
  substituted at draw time.
- The lookup functions — name to identifier, identifier to name, key name to scancode,
  scancode to key, and both directions of the binding query — all contracted in the
  implementation twin.
- **`ForAllActionKeys`** — visit each bound slot of one action, skipping the unbound, with an
  optional early stop. The shape exists so callers need not know there are three slots.
- **`ConsoleBindCmds`** — the separate map from scancode to a console line, with bind,
  unbind, execute, clear and save.
- **`key_binding_registrator`** — the script registration hook, implemented in
  [`key_binding_registrator_script.cpp`](key_binding_registrator_script.cpp.md).
