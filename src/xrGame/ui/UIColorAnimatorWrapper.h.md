# src/xrGame/ui/UIColorAnimatorWrapper.h

> Declares the playback head that drives a widget's colour from a named light-animation curve on the screen's own clock.

**Needs** — [`UIColorAnimatorWrapper.cpp`](UIColorAnimatorWrapper.cpp.md) · [`xrEngine/LightAnimLibrary.h`](../../xrEngine/LightAnimLibrary.h.md)
**Used by** — [`UIColorAnimatorWrapper.cpp`](UIColorAnimatorWrapper.cpp.md)
**Tier floor** — T3: a clock, a curve lookup and a packed colour; nothing device-facing.

## Purpose

Declares the surface implemented in [`UIColorAnimatorWrapper.cpp`](UIColorAnimatorWrapper.cpp.md).
The animation library shipped with the engine is built for world lighting, which is sampled
every frame; screens are not. The wrapper exists so that a screen can sample the same curves
against elapsed wall time it accumulates itself.

## Exported units

- **`CUIColorAnimatorWrapper`** — one playback head bound to one named curve.
  - construction from a curve name, or from a destination colour slot, or bare;
  - `SetColorAnimation` / `SetColorToModify` — rebind curve or destination;
  - `Update` — advance and sample; the screen must call this from its own update;
  - `Cyclic`, `Reverese`, `Reset`, `GoToEnd` — playback mode and seek;
  - `GetColor`, `LastFrame`, `TotalFrames`, `Done` / `SetDone` — readback.

**Notes** — the destination is an *aliased colour slot*: the head writes the sampled colour
through a reference the screen handed it, so a widget's colour field updates without the
screen copying it each tick. A rebuild that cannot alias a field gives the head an explicit
sink instead — a callback or an observed value — and the calling screen stops being
responsible for the copy. `Reverese` is spelled that way in the shipped surface.
