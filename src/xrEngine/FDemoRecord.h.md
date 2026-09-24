# src/xrEngine/FDemoRecord.h

> Declares the demo-recording camera.

**Needs** — [`Effector.h`](Effector.h.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`GameFont.h`](GameFont.h.md) · [`pure.h`](pure.h.md)
**Used by** — [`FDemoRecord.cpp`](FDemoRecord.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`FDemoRecord.cpp`](FDemoRecord.cpp.md).

## Exported units

- **`DemoRecorder`** — simultaneously a camera effector (it overrides the camera), an input receiver (it takes the keyboard) and a render handler (it draws its own overlay). Constructed with the output file name and a lifetime defaulting to one hour.
- **`apply`** — the per-frame camera step.
- **The input entry points** — key press, hold and release; mouse move and hold; gamepad press, hold, release and attitude change. All of them either act or pass through to the game, depending on one toggle.
- **`set_global_position` / `get_global_position`** — a one-slot channel the game uses to teleport the recorder and to read where it is.
- **`redirect_input_to_level`** — the pass-through toggle, public because the game reads it.
- **`on_render`** — draws the help overlay.

**Notes** — The four speed steps are an enumeration rather than a continuous value because the authored configuration provides exactly four translation and four rotation speeds. A rebuild could interpolate, but the shipped configuration only defines the four.
