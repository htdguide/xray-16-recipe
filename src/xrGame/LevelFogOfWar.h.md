# src/xrGame/LevelFogOfWar.h

> Declares the per-level exploration grid and the manager that keeps one per visited level, implemented in [`LevelFogOfWar.cpp`](LevelFogOfWar.cpp.md).

**Needs** — [`ui/UIWindow.h`](../xrUICore/Windows/UIWindow.h.md) · [`alife_abstract_registry.h`](alife_abstract_registry.h.md)
**Used by** — [`LevelFogOfWar.cpp`](LevelFogOfWar.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CLevelFogOfWar` and `CFogOfWarMngr`. Substance is in
[`LevelFogOfWar.cpp`](LevelFogOfWar.cpp.md).

The shape it fixes is an unusual double role: the exploration grid is **both** a serializable
record in the alife registry and a child window of the map screen. One object is therefore
saved into the campaign's persistent state and drawn as part of an interface layout. A
rebuild should separate the two — the grid is data and the drawing is a view of it — but must
keep the consequence, which is that exploration survives leaving a level because it is
registry state rather than screen state.

Exported units:

- `CLevelFogOfWar` — the grid: level name, world bounds, row and column counts, one bit per
  cell, plus the material pass and geometry stream it draws with.
- `Init` — size the grid from the level's authored bounds.
- `Open(position)` — reveal the cells near a world position. This is what walking calls.
- `Open(row, col, mask)` — set one cell, bounds-checked.
- `Draw` — the map-screen overlay.
- `GetTexUVLT` — a cell's texture origin, selecting the open or closed half of one material.
- `ConvertRealToLocal` / `ConvertLocalToReal` — the world/cell mapping, for a point and a
  rectangle.
- `save` / `load` — the saved form, with the stored geometry verified against configuration
  on load.
- `CFogOfWarMngr` — one grid per visited level, held in the alife registry.
- `GetFogOfWar` — the grid for a named level, created on first request; nothing outside
  single player.
