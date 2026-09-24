# src/xrUICore/ScrollBar/UIScrollBar.h

> Declares the scroll bar implemented in [`UIScrollBar.cpp`](UIScrollBar.cpp.md), including the protected surface its fixed-geometry variant overrides.

**Needs** — [`UIScrollBar.cpp`](UIScrollBar.cpp.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md)
**Used by** — [`UIDragDropListEx.cpp`](../../xrGame/ui/UIDragDropListEx.cpp.md) · [`UIComboBox.cpp`](../ComboBox/UIComboBox.cpp.md) · [`UIListBox.cpp`](../ListBox/UIListBox.cpp.md) · [`UIListWnd.cpp`](../ListWnd/UIListWnd.cpp.md) · [`UIListWnd.h`](../ListWnd/UIListWnd.h.md) · [`UIListWnd_inline.h`](../ListWnd/UIListWnd_inline.h.md) · [`UIFixedScrollBar.cpp`](UIFixedScrollBar.cpp.md) · [`UIFixedScrollBar.h`](UIFixedScrollBar.h.md) · [`UIScrollBar.cpp`](UIScrollBar.cpp.md) · [`UIScrollView.cpp`](../ScrollView/UIScrollView.cpp.md) · [`UIScrollView.h`](../ScrollView/UIScrollView.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIScrollBar.cpp`](UIScrollBar.cpp.md). It matters beyond
being a declaration for one reason: it fixes which parts of the bar are *overridable*, and
that set is exactly the seam [`UIFixedScrollBar`](UIFixedScrollBar.h.md) cuts along —
layout, clamping, and the pixels-to-position conversion. Everything else, including the whole
range model and the step primitives, is shared.

## Exported units

The scroll model:

- `SetRange` / `GetRange` / `GetMinRange` / `GetMaxRange` — the integer position range.
- `SetPageSize` / `GetPageSize` — how much of that range the viewer shows at once.
- `SetStepSize` / `GetStepSize` — one stepper click.
- `SetScrollPos` / `GetScrollPos` — the position, always clamped into range on write.
- `TryScrollInc` / `TryScrollDec` — step and notify the owner if anything moved.
- `Refresh` — re-notify the owner from the thumb's current position.
- `Reset`.

Construction and lifecycle:

- `InitScrollBar(position, length, horizontal, profile)` — build from a named profile of the
  shipped scroll-bar layout; returns whether every asset was found in its modern form.
- `SetEnabled` / `GetEnabled` — the hard off-switch: a disabled bar cannot be shown.
- `Show` / `Enable` — both refuse while the hard switch is off.
- `SetWidth` / `SetHeight` — resizing recomputes the track length.

Input:

- `OnMouseAction` / `OnMouseDown` / `OnMouseUp` / `OnKeyboardAction` — wheel, press, release
  and the held-button auto-repeat.
- `OnMouseDownEx` — the five-region hit test, factored out so the repeat can re-run it.
- `SendMessage` — receives the steppers' clicks and the thumb's drag reports.

Overridable internals (the variant seam):

- `UpdateScrollBar` — recompute thumb size and position.
- `ClampByViewRect` — push the thumb back inside the track.
- `SetPosScrollFromView` — drag pixels back into a position.
- `PosViewFromScroll` — position into drag pixels. Not virtual: both variants share it.
- `ScrollSize` — the number of distinct positions, floored at one.
- `IsRelevant` — is there anywhere to scroll.
