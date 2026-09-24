# src/xrGame/ui/UIFrags.cpp

> A single-column multiplayer scoreboard panel: one statistics table dressed in a three-piece vertical frame whose middle section tiles to the table's height. Excluded from the build.

**Needs** — [`UIFrags.h`](UIFrags.h.md) · [`UIStats.h`](UIStats.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIFrags.h`](UIFrags.h.md)
**Tier floor** — T3.

## Purpose

The deathmatch scoreboard: a player statistics table with a decorative frame around it. It is
**commented out of the build** and nothing references it; the surviving scoreboard is built
directly from the statistics table. It is described here because it records one decision the
surviving code still relies on, namely how a panel of variable row count is dressed.

## State

```text
RECORD FragsPanel EXTENDS Window
  cap_top    : Picture   # fixed-height top cap
  body       : Picture   # tiled vertically `count` times
  cap_bottom : Picture   # fixed-height bottom cap
  table      : StatsTable
```

## `Init`

**Contract** — build the statistics table from the layout at the given element path with
team index zero (no teams), then dress the frame from a second, separate element path.

**Notes** — two paths, not one. The frame is authored somewhere else in the same document
than the table, because the same frame dressing is shared between the one-column and the
two-column scoreboards and the tables are not.

## The three-piece frame

**Contract** — the panel is a vertical stretchable frame built by hand rather than by the
toolkit's frame widgets: a top cap, a bottom cap, and a middle picture **tiled `count` times
vertically**, where `count` is an attribute on the layout element.

**Notes** — chapter 15 offers two stretchable frame shapes, the three-segment line and the
nine-slice window, and neither takes a tile count from data. This panel needs one because the
scoreboard's height is a whole number of rows and the frame's middle texture is exactly one
row tall: tiling by count makes the frame's seams land on the row boundaries. Stretching
instead would blur them. A rebuild that uses a generic nine-slice here will not reproduce the
shipped look.

The attribute is read from a buffer that has already been overwritten by the path
concatenation just above it, so the count is looked up on a sub-element path rather than on
the frame element the author would expect. It reads as an accident; the default of one is
what the shipped data gets. Recorded as unrecovered intent.
