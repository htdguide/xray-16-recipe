# src/xrUICore/ProgressBar/UIProgressBar.h

> Declares the bar — a range, a position that chases its target, six fill directions, and an optional colour that interpolates with the fill.

**Needs** — [`UIProgressBar.cpp`](UIProgressBar.cpp.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md)
**Used by** — [`UIActorStateInfo.cpp`](../../xrGame/ui/UIActorStateInfo.cpp.md) · [`UICellItem.cpp`](../../xrGame/ui/UICellItem.cpp.md) · [`UIDemoPlayControl.cpp`](../../xrGame/ui/UIDemoPlayControl.cpp.md) · [`UIDragDropListEx.cpp`](../../xrGame/ui/UIDragDropListEx.cpp.md) · [`UIFactionWarWnd.cpp`](../../xrGame/ui/UIFactionWarWnd.cpp.md) · [`UIHudStatesWnd.cpp`](../../xrGame/ui/UIHudStatesWnd.cpp.md) · [`UIMotionIcon.h`](../../xrGame/ui/UIMotionIcon.h.md) · [`UIRankFaction.cpp`](../../xrGame/ui/UIRankFaction.cpp.md) · [`UIDoubleProgressBar.cpp`](UIDoubleProgressBar.cpp.md) · [`UIDoubleProgressBar.h`](UIDoubleProgressBar.h.md) · [`UIProgressBar.cpp`](UIProgressBar.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIProgressBar.cpp`](UIProgressBar.cpp.md).

Two decisions live in the shape of the record. The position is a *pair* — where the bar is
drawn and where it is heading — so that every bar in the game animates toward its value rather
than snapping; a bar with no inertia still goes through the same path. And the fill is not a
scaled quad: it is a full-size quad clipped to a computed rectangle, which is what lets the
same texture serve every fill fraction without stretching.

## Exported units

- `CUIProgressBar` — the bar.
- `EOrientMode` — the six fill directions: left-to-right, bottom-to-top, right-to-left,
  top-to-bottom, horizontally from the centre outward, vertically from the centre outward.
  **Frozen**: the XML names them by index.
- `InitProgressBar(pos, size, mode)`.
- `SetRange(min, max)` / `GetRange_min` / `GetRange_max`.
- `SetProgressPos(value)` — set the target; the drawn position chases it.
- `ForceSetProgressPos(value)` — set both at once, no animation.
- `GetProgressPos` — the *target*, not the drawn position.
- `ShowBackground` / `IsShownBackground`.
- `UseGradient` — whether colour interpolates or is the single maximum colour.
- `m_UIProgressItem` / `m_UIBackgroundItem` — the fill and the backdrop, public so the XML
  layer and the double bar can reach them.

**Notes** — the bar is disabled at construction, so it never takes a hit test. A progress bar
is decoration; it is never a control.
