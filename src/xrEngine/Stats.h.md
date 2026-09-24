# src/xrEngine/Stats.h

> Declares the statistics overlay and the four spare timers every developer reaches for.

**Needs** — [`Stats.cpp`](Stats.cpp.md) · [`StatGraph.h`](StatGraph.h.md) · [`pure.h`](pure.h.md)
**Used by** — [`Device_create.cpp`](Device_create.cpp.md) · [`Stats.cpp`](Stats.cpp.md) · [`device.cpp`](device.cpp.md) · [`device.h`](device.h.md)
**Tier floor** — T2: text layout and a graph, once per frame

## Purpose

Declares the surface implemented in [`Stats.cpp`](Stats.cpp.md).

Exported units:

- `CStats` — the overlay. Owns two fonts and the frame-rate graph, collects error lines out
  of the log, and calls every subsystem's own statistics dump in a fixed order.
- `Show` — the per-frame text pass, run from the render phase.
- `OnRender` — the per-frame *world-space* debug pass, currently the sound-source
  visualization.
- `OnDeviceCreate` / `OnDeviceDestroy` — font and graph lifetime, and the log tap.
- `gTestTimer0..3` — four unowned timers, started and ended by the overlay every frame and
  printed by it, so a developer can bracket any code with one and read the result on screen
  without adding a counter. They ship in every build.
- The sound-visualization flags: whether to draw sources at all, their minimum, maximum and
  AI-audible radii, and their names and owning objects.
