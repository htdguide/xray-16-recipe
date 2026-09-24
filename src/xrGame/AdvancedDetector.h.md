# src/xrGame/AdvancedDetector.h

> Declares the directional artefact detector implemented in [`AdvancedDetector.cpp`](AdvancedDetector.cpp.md).

**Needs** — [`CustomDetector.h`](CustomDetector.h.md)
**Used by** — [`AdvancedDetector.cpp`](AdvancedDetector.cpp.md) · [`torch_script.cpp`](torch_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CAdvancedDetector`, the middle grade of the three artefact detectors: it shows
direction as well as proximity. Substance is in
[`AdvancedDetector.cpp`](AdvancedDetector.cpp.md).

Exported units:

- `CAdvancedDetector` — the device. Its only stored distinction from the base detector is
  the artefact rank it can see, set to two at construction.
- `UpdateAf` — find the nearest unowned artefact, point the needle at it, and beep.
- `CreateUI` / `ui` — build and reach the presentation object that owns the needle bone.
- `on_a_hud_attach` / `on_b_hud_detach` — install and remove the needle bone callback when
  the device enters and leaves the player's hands.
