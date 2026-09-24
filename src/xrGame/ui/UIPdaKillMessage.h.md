# src/xrGame/ui/UIPdaKillMessage.h

> Declares one kill report: killer, weapon, victim and circumstance, laid out as a single line
> of alternating coloured names and icons.

**Needs** — [`UIPdaKillMessage.cpp`](UIPdaKillMessage.cpp.md) · [`KillMessageStruct.h`](KillMessageStruct.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIGameLog.cpp`](UIGameLog.cpp.md) · [`UIGameLog.h`](UIGameLog.h.md) · [`UIPdaKillMessage.cpp`](UIPdaKillMessage.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIPdaKillMessage.cpp`](UIPdaKillMessage.cpp.md).

## Exported units

- **The kill report row** — four widgets on one line inside a container that fades itself.
- `Init` — fill the row from a kill record and a font, and size it to what was actually placed.
- `InitText` / `InitIcon` — place one element and report the width it consumed; each reports
  zero for an element the record left empty.
