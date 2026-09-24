# src/xrUICore/arrow/ui_arrow.h

> Declares the needle widget — a static whose rotation angle tracks a normalized value, implemented in [`ui_arrow.cpp`](ui_arrow.cpp.md).

**Needs** — [`ui_arrow.cpp`](ui_arrow.cpp.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md)
**Used by** — [`UIActorStateInfo.cpp`](../../xrGame/ui/UIActorStateInfo.cpp.md) · [`UIHudStatesWnd.cpp`](../../xrGame/ui/UIHudStatesWnd.cpp.md) · [`ui_arrow.cpp`](ui_arrow.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`ui_arrow.cpp`](ui_arrow.cpp.md): the dial needle used by
the radiation detector, the compass and the vehicle gauges. It is a static with one extra
idea — the widget's rotation is a function of a value in the unit interval, and the value is
chased rather than jumped to.

## Exported units

- `UI_Arrow` — the needle.
- `init_from_xml` — attaches to a parent, reads the static's own appearance, then reads the
  four dial parameters: start angle, end angle, angular speed, and sweep direction.
- `SetNewValue` — the chasing entry point: request a target, get one frame of movement.
- `SetPos` / `GetPos` — set or read the value immediately, with no chasing.
