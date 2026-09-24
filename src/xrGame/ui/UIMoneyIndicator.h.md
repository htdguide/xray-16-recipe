# src/xrGame/ui/UIMoneyIndicator.h

> Declares the multiplayer money readout: a total, a transient change notice, and a list of
> recent bonus awards.

**Needs** — [`UIMoneyIndicator.cpp`](UIMoneyIndicator.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIGameAHunt.cpp`](../UIGameAHunt.cpp.md) · [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`UIGameDM.cpp`](../UIGameDM.cpp.md) · [`UIGameTDM.cpp`](../UIGameTDM.cpp.md) · [`UIMoneyIndicator.cpp`](UIMoneyIndicator.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMoneyIndicator.cpp`](UIMoneyIndicator.cpp.md).

## Exported units

- **The money readout** — a window with a backing plate, a total, a change notice and a
  scrolling bonus list.
- `InitFromXML` — build from the overlay's layout document; declines when the document has no
  money panel, which is how single player gets none.
- `SetMoneyAmount` — the total, pre-formatted by the caller.
- `SetMoneyChange` — show a change and restart its fade.
- `AddBonusMoney` — append one structured award to the bonus list.
