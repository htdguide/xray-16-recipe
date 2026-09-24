# src/xrGame/ui/UIPdaMsgListItem.h

> Declares one row of the on-screen message log: an icon, a timestamp, a caption and a body,
> inside a container that fades itself out.

**Needs** — [`UIPdaMsgListItem.cpp`](UIPdaMsgListItem.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIGameLog.cpp`](UIGameLog.cpp.md) · [`UIGameLog.h`](UIGameLog.h.md) · [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIMessagesWindow.cpp`](UIMessagesWindow.cpp.md) · [`UIPdaMsgListItem.cpp`](UIPdaMsgListItem.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIPdaMsgListItem.cpp`](UIPdaMsgListItem.cpp.md). The four
widgets are public because the row is filled in by whoever appends it — see
[`UIMessagesWindow`](UIMessagesWindow.cpp.md) — rather than by a method here.

## Exported units

- **The message row** — four pictures in a self-fading container.
- `InitPdaMsgListItem` — size the row and build whichever of its four widgets the shared row
  layout document defines.
- `SetFont` — apply one font to all three text widgets at once.
