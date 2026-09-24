# src/xrGame/ui/UIFrags2.cpp

> The team scoreboard: the same frame with two statistics tables side by side, the second one's column origin authored as a single number. Excluded from the build.

**Needs** — [`UIFrags2.h`](UIFrags2.h.md) · [`UIFrags.h`](UIFrags.h.md) · [`UIStats.h`](UIStats.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIFrags2.h`](UIFrags2.h.md)
**Tier floor** — T3.

## Purpose

The team-game scoreboard. It extends the single-column panel with a second table and lays the
two out as two columns inside one frame. Like its base it is **commented out of the build**.

## `Init`

**Contract** — dress the frame first, then build both tables — team one and team two — from
the *same* layout path, distinguished only by the team index passed to the table builder.
Then place them:

```text
FUNCTION lay_out_two_tables(layout, path)
  dress the frame
  header1 := table1.build(layout, path, team = 1)
  header2 := table2.build(layout, path, team = 2)
  x2 := layout.attribute(path, "x2")         # required; zero is rejected
  table2.position := (x2, table2.position.y + 3)
  header2.position.x := header2.position.x + table2.position.x
  table1.position.y := table1.position.y + 3
  header1.position.x := header1.position.x + table1.position.x
```

**Invariants** — the table builder returns a *header* window separate from the table itself,
and the header is positioned independently. Both must be shifted by their table's origin,
because the header is authored in the table's local coordinates but attached to the panel.

**Notes** — one authored number, the second column's left edge, is all that distinguishes the
two-column layout from the one-column one; everything else about a table is shared. A zero is
rejected outright rather than defaulted, because a missing attribute would silently stack the
two teams on top of each other.

The three-unit downward nudge applied to both tables is the gap between the frame's top cap
and the first row. It is applied in code rather than authored, which means the layout cannot
change it — recorded as an arbitrary-looking constant with no recoverable justification
beyond matching the frame texture.
