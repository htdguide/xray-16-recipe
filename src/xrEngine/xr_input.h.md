# src/xrEngine/xr_input.h

> Declares the unified key space, the controller state record and the input layer; the substance is in [`xr_input.cpp`](xr_input.cpp.md).

**Needs** — [`xr_input.cpp`](xr_input.cpp.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`pure.h`](pure.h.md) · [`xrCore/_vector2.h`](../xrCore/_vector2.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`engine_impl.cpp`](../editors/xrWeatherEngine/engine_impl.cpp.md) · [`Device_Initialize.cpp`](Device_Initialize.cpp.md) · [`Device_destroy.cpp`](Device_destroy.cpp.md) · [`Device_mode.cpp`](Device_mode.cpp.md) · [`Engine.cpp`](Engine.cpp.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`IInputReceiver.cpp`](IInputReceiver.cpp.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`Stats.cpp`](Stats.cpp.md) · [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) · [`device.cpp`](device.cpp.md) · [`edit_actions.cpp`](edit_actions.cpp.md) · [`editor_base_input.cpp`](editor_base_input.cpp.md) · [`line_edit_control.cpp`](line_edit_control.cpp.md) · _and 15 more_
**Tier floor** — T2.

## Purpose

Declares the surface described in [`xr_input.cpp`](xr_input.cpp.md), and *defines the key
space*, which is the part a rebuild must reproduce faithfully because bindings are stored
against it.

Exported units:

- **The key space** — three enumerations laid end to end. Mouse buttons begin one past the
  last keyboard scancode; controller buttons one past the last mouse button; controller
  axes one past the last controller button. Each range begins with a marker value meaning
  "none of this kind". Each carries a count derived from its own bounds rather than written
  out, so adding an entry cannot desynchronise the count.
- **`ControllerAxisState`** — a two-dimensional axis value plus its magnitude, kept
  together because every consumer wants both and recomputing the length is wasteful. Stored
  as an overlay of a vector and two named components.
- **`ControllerState`** — the four axes (addressable both by name and by index over the
  same storage), the button set, the gyroscope reading, and which physical controller is
  active.
- **`CInput`** — the input layer: the receiver stack (`iCapture`, `iRelease`, `CurrentIR`),
  the polled queries, cursor and grab control, the text-input bracket, the key-map-change
  signal registry, the per-frame pass, and the rumble call.
- **`KeyMapChanged`** — a signal, declared here, broadcast when the platform's keyboard
  layout changes. Watchers re-resolve their displayed key names; the *bindings* do not
  change, because they are stored by scancode.
- The process-wide input tuning values.

## Notes

The axis enumeration is guarded by a compile-time check against the windowing layer's own
axis count, so that a new axis appearing upstream is a build failure rather than a silent
mismatch. The named-and-indexed overlay of the axis record is guarded the same way. Both
are the right instinct expressed in the only mechanism available; a rebuild should keep the
*checks* and can express them however it likes.

The constants naming which platforms have global mouse position and capture are a
capability probe done at build time. A rebuild should probe at run time or require the
capability.
