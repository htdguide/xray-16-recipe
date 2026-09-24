# src/xrUICore/ScrollBar/UIScrollBox.h

> Declares the scroll bar's draggable thumb, implemented in [`UIScrollBox.cpp`](UIScrollBox.cpp.md).

**Needs** — [`UIScrollBox.cpp`](UIScrollBox.cpp.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md)
**Used by** — [`UIFixedScrollBar.cpp`](UIFixedScrollBar.cpp.md) · [`UIScrollBar.cpp`](UIScrollBar.cpp.md) · [`UIScrollBox.cpp`](UIScrollBox.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIScrollBox.cpp`](UIScrollBox.cpp.md). The thumb is a
three-segment stretchable line — the same drawer the track uses — with one behaviour added
and nothing else: it can be dragged. It owns no position model; it moves itself and tells its
parent.

## Exported units

- `CUIScrollBox` — the thumb.
- `OnMouseAction` — the entire drag protocol.
