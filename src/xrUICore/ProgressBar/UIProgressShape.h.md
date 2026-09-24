# src/xrUICore/ProgressBar/UIProgressShape.h

> Declares the circular progress indicator implemented in [`UIProgressShape.cpp`](UIProgressShape.cpp.md).

**Needs** — [`UIProgressShape.cpp`](UIProgressShape.cpp.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md)
**Used by** — [`Missile.cpp`](../../xrGame/Missile.cpp.md) · [`UIGameAHunt.cpp`](../../xrGame/UIGameAHunt.cpp.md) · [`UIGameCTA.cpp`](../../xrGame/UIGameCTA.cpp.md) · [`UIActorStateInfo.cpp`](../../xrGame/ui/UIActorStateInfo.cpp.md) · [`UIHudStatesWnd.cpp`](../../xrGame/ui/UIHudStatesWnd.cpp.md) · [`UIMotionIcon.h`](../../xrGame/ui/UIMotionIcon.h.md) · [`UIProgressShape.cpp`](UIProgressShape.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the ring-shaped progress widget implemented in
[`UIProgressShape.cpp`](UIProgressShape.cpp.md). The file records one decision of its own:
the shape does **not** share the linear progress bar's range model. A linear bar owns a
minimum, a maximum and a position; the shape owns only a fraction, and the caller does the
division. The source carries a standing note that this ought to be unified; a rebuild that
unifies them must keep the fractional entry point, because the XML vocabulary and the script
surface both set a shape by fraction.

## Exported units

- `CUIProgressShape` — a [static](../Static/UIStatic.h.md) that draws a fanned ring.
- `SetPos(fraction)` and `SetPos(value, maximum)` — the two ways to state progress; the second
  additionally writes the raw value as text when text is enabled.
- `SetTextVisible` — whether the numeric value is drawn inside the ring.
- `Draw` — emits the fan.
- The configurable shape: sector count (eight by default), sweep direction, start and end
  angle (a full turn by default), whether sector opacity is smoothed or stepped, and two
  optional child statics — one supplying the ring art and its texture rectangle, one drawn
  as a backdrop.

**Notes** — every one of those knobs is written by the XML reader, which is declared a friend
so it can reach them directly. In a rebuild that is a construction-time configuration record
handed to the widget, not privileged access.
