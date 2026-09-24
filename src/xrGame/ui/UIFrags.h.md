# src/xrGame/ui/UIFrags.h

> Declares the single-column scoreboard panel. Excluded from the build.

**Needs** — [`UIFrags.cpp`](UIFrags.cpp.md) · [`UIStats.h`](UIStats.h.md)
**Used by** — [`UIFrags.cpp`](UIFrags.cpp.md) · [`UIFrags2.cpp`](UIFrags2.cpp.md) · [`UIFrags2.h`](UIFrags2.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIFrags.cpp`](UIFrags.cpp.md). Both files are commented
out of the build; a rebuild may drop them.

## Exported units

- **`CUIFrags`** — a statistics table plus a three-piece vertical frame.
  - `Init` — build the table from one layout path and the frame from another.
  - `InitBackground` — dress the frame; separated so the two-column variant can call it
    before laying out its two tables.
