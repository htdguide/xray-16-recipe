# src/xrGame/ui/UIFrameLine.h

> Declares the three-sprite stretchable line. Excluded from the build.

**Needs** — [`UIFrameLine.cpp`](UIFrameLine.cpp.md) · [`UIStaticItem.h`](../../xrUICore/Static/UIStaticItem.h.md)
**Used by** — [`UIFrameLine.cpp`](UIFrameLine.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`UIFrameLine.cpp`](UIFrameLine.cpp.md). Both files are
commented out of the build; chapter 15's frame-line widget supersedes them.

## Exported units

- **`CUIFrameLine`** — three sprites indexed by role: first (left or top), second (right or
  bottom), back (the tiled middle). The role names are axis-neutral because one class serves
  both orientations.
  - `InitFrameLine` — origin, length, orientation and alignment in one call.
  - `InitTexture` — resolve the three sprites from one stem plus the frozen suffixes.
  - `SetPos` / `SetSize` / `SetOrientation` — each invalidates the cached geometry.
  - `set_parent_wnd_size` — supply the owner's box, required before drawing when stretching.
  - `bStretchTexture` — public, because callers flip it directly after construction.
  - `SetColor` — one tint across all three pieces.
  - `Render` — recompute if stale, then submit.
