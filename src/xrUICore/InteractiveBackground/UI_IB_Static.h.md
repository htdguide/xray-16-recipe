# src/xrUICore/InteractiveBackground/UI_IB_Static.h

> Declares the four-state background whose slots are plain textured quads, and adds the two quad-only adjustments the generic container cannot express.

**Needs** — [`UI_IB_Static.cpp`](UI_IB_Static.cpp.md) · [`UIInteractiveBackground.h`](UIInteractiveBackground.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md)
**Used by** — [`UI3tButton.cpp`](../Buttons/UI3tButton.cpp.md) · [`UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`UIInteractiveBackground.h`](UIInteractiveBackground.h.md) · [`UI_IB_Static.cpp`](UI_IB_Static.cpp.md) · [`UITrackBar.cpp`](../TrackBar/UITrackBar.cpp.md) · [`UITrackBar.h`](../TrackBar/UITrackBar.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UI_IB_Static.cpp`](UI_IB_Static.cpp.md). The generic
container knows nothing about textures beyond loading them; a quad background additionally
has a texture offset and a stretch flag, and both must be applied to all four slots at once
or a state change would change the look in a way nobody asked for.

## Exported units

- `CUI_IB_Static` — the quad-backed four-state background.
- `SetTextureOffset(x, y)` — applies to every slot.
- `SetStretchTexture(on)` — applies to every slot.

**Notes** — the sibling stretched-line form is not a class at all, only a name for the generic
container over the stretched-line widget, declared at the bottom of
[`UIInteractiveBackground.h`](UIInteractiveBackground.h.md). The asymmetry exists only
because the line form needs no extra methods.
