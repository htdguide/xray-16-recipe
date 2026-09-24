# src/xrUICore/ScrollBar/UIFixedScrollBar.h

> Declares the fixed-geometry scroll bar implemented in [`UIFixedScrollBar.cpp`](UIFixedScrollBar.cpp.md) — a scroll bar whose size comes from its art, not from its owner.

**Needs** — [`UIFixedScrollBar.cpp`](UIFixedScrollBar.cpp.md) · [`UIScrollBar.h`](UIScrollBar.h.md)
**Used by** — [`UILogsWnd.cpp`](../../xrGame/ui/UILogsWnd.cpp.md) · [`UIMapWnd.cpp`](../../xrGame/ui/UIMapWnd.cpp.md) · [`UIFixedScrollBar.cpp`](UIFixedScrollBar.cpp.md) · [`UIScrollView.cpp`](../ScrollView/UIScrollView.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the variant implemented in [`UIFixedScrollBar.cpp`](UIFixedScrollBar.cpp.md). The
declaration states the two decisions that define the variant.

**Resizing is refused.** Both size setters are overridden to do nothing. The bar's dimensions
come from its layout profile and the owner may not stretch it — which is the whole meaning of
"fixed". A rebuild must keep the refusal, not merely avoid resizing: the scroll view calls
the setters unconditionally.

**The thumb is a button, not a stretchable line.** It shadows the base class's thumb field
with a differently typed one, so every method that touches the thumb must be overridden —
which is why the override list is as long as it is. That is a cost of the C++ shadowing
trick, not a design; a rebuild with one thumb interface and two implementations needs far
fewer overrides.

## Exported units

- `CUIFixedScrollBar` — the variant.
- `InitScrollBar(position, horizontal, profile)` — note the missing length parameter, the
  signature difference that expresses "fixed". Returns false when the profile lacks the
  three-segment track art, which is the scroll view's signal to fall back to the ordinary bar.
- `SetWidth` / `SetHeight` — deliberately inert.
- `UpdateScrollBar` / `ClampByViewRect` / `SetPosScrollFromView` — the layout and mapping
  overrides, each adding a per-axis inset between the thumb and the steppers.
- `OnMouseAction` / `OnMouseDown` / `OnMouseDownEx` / `OnMouseUp` / `OnKeyboardAction` /
  `SendMessage` / `Draw` — re-implemented against the button thumb.
