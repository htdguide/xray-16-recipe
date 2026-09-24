# src/xrEngine/IInputReceiver.h

> The interface anything that wants the keyboard, mouse or gamepad implements — plus every input tuning value the player can set.

**Needs** — [`xr_level_controller.h`](xr_level_controller.h.md) · [`xr_input.h`](xr_input.h.md)
**Used by** — [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`FDemoRecord.h`](FDemoRecord.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IInputReceiver.cpp`](IInputReceiver.cpp.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`editor_base.h`](editor_base.h.md) · [`xr_input.cpp`](xr_input.cpp.md) · [`xr_input.h`](xr_input.h.md) · [`MainMenu.h`](../xrGame/MainMenu.h.md) · [`Spectator.cpp`](../xrGame/Spectator.cpp.md) · [`Spectator.h`](../xrGame/Spectator.h.md) · [`UIGameTutorial.h`](../xrGame/ui/UIGameTutorial.h.md)
**Tier floor** — T2: an interface plus a set of console-settable scalars.

## Purpose

Input is delivered to exactly one receiver at a time — the top of a stack. The console, the main menu, the game level, the demo recorder and the debug overlay are all receivers, and pushing one onto the stack is how a subsystem takes the keyboard. This header is what a receiver must provide and what it may ask.

Everything here is an *event* interface, not a polling one, with one polling escape (`is_key_down`) for code that needs to ask about a modifier without having received an event for it.

## `InputReceiver`

**Contract** — A receiver implements only the events it cares about; every entry point has an empty default. The set is:

- **Mouse** — press, release, hold, wheel (two axes), and relative motion.
- **Keyboard** — press, release, hold, and separately *text input*, which delivers composed characters rather than keys. The separation is required: a key event carries a scancode that is independent of layout, and text input carries what the player actually typed under their layout and input method. The console needs both — bindings by scancode, typing by character.
- **Gamepad** — press, release and hold, each accompanied by the current axis state, plus a device-orientation change for controllers with motion sensors.

A **hold** event is delivered every frame a key remains down, which is what lets movement code accumulate a per-frame contribution without tracking key state itself.

## `capture` / `release`

**Contract** — Push this receiver onto the input stack, or remove it. Implemented in [`IInputReceiver.cpp`](IInputReceiver.cpp.md).

## `on_activate` / `on_deactivate`

**Contract** — Called when this receiver reaches or leaves the top of the stack. The deactivation default is the load-bearing one — see [`IInputReceiver.cpp`](IInputReceiver.cpp.md).

## `is_key_down`

**Contract** — Poll whether a key, mouse button or gamepad control is currently held. Reads the input layer's own state rather than anything the receiver tracked.

## Input tuning values

```text
mouse_sensitivity, mouse_sensitivity_scale   # the player's setting, and a game-applied multiplier
mouse_invert                                 # per-axis inversion flags

controller_stick_sensitivity_x / _y          # per-axis, because vertical look is slower
controller_stick_sensitivity_scale
controller_stick_inner_dead_zone             # deflection below this reads as centred
controller_stick_outer_dead_zone             # deflection above this reads as fully deflected
controller_stick_angular_dead_zone           # snaps near-axial directions onto the axis
controller_sensor_sensitivity                # motion-sensor aiming
controller_sensor_dead_zone
controller_cursor_autohide_time              # seconds before an idle pointer hides in menus
controller_flags                             # invert X, invert Y, enable motion sensors
```

**Notes** — Two separate sensitivity values per input, a *setting* and a *scale*. The setting is the player's; the scale is the game's, applied transiently when a weapon is zoomed so that aiming slows without touching what the player configured. Multiplying them at the use site rather than storing one combined value is what keeps the player's setting recoverable.

**Notes** — Three dead zones on a stick, and they do different jobs. The inner one removes drift at rest. The outer one lets a worn stick still reach full deflection. The angular one snaps a nearly-horizontal push to exactly horizontal, which is what makes strafing and turning feel crisp rather than always slightly diagonal. A rebuild that implements only the inner dead zone will feel noticeably worse.
