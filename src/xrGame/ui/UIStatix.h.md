# src/xrGame/ui/UIStatix.h

> Declares the selectable picture: a picture that answers a click like a button and pulses while
> selected.

**Needs** — [`UIStatix.cpp`](UIStatix.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UISkinSelector.cpp`](UISkinSelector.cpp.md) · [`UISpawnWnd.cpp`](UISpawnWnd.cpp.md) · [`UIStatix.cpp`](UIStatix.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIStatix.cpp`](UIStatix.cpp.md). The multiplayer team and skin
pickers both need a large picture that behaves as a mutually-exclusive choice; this is that widget.

Exported units:

- `CUIStatix` — the selectable picture.
- `SetSelectedState(state)` / `GetSelectedState()` — the selection, which drives the pulse.
- `OnMouseDown` — the click-to-notify behaviour.
