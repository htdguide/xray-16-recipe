# src/xrGame/ui/UIRankIndicator.cpp

> The multiplayer rank badge: ten pictures built up front, one attached at a time, with team and
> rank folded into a single index.

**Needs** — [`UIRankIndicator.h`](UIRankIndicator.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIRankIndicator.h`](UIRankIndicator.h.md)
**Tier floor** — T3.

## Purpose

Shows which rank the player has reached, in their team's colours. The decision here is the
*representation*: rather than one picture whose texture is swapped, there are ten pictures and
the badge attaches one. That costs ten widgets and buys the ability for each badge to differ in
size, position and animation — which the shipped layouts use.

## State

```text
RECORD RankBadge EXTENDS Window
  ranks   : list<Static>   # ten, elements "rank_wnd:rank_0".."rank_9"; built but unattached
  current : int            # which one is attached, or none
```

**Invariants**

- **The ten pictures are owned by the badge and attached only one at a time.** Nine of them are
  therefore alive and parentless at any moment, which is why they are deleted explicitly rather
  than by the tree.
- The index is `rank + team * 5`: the first five badges are one team's ranks and the second five
  are the other's. Five is half the fixed ten, written as such, so the two are locked together —
  changing the badge count changes the ranks per team.
- The backing picture is attached permanently and is the only child when no rank is set.

## `InitFromXml`

**Contract** — apply the `rank_wnd` element, declining if the document has none. Build all ten
rank pictures from their numbered elements, build the backing picture and attach it. The ten are
not attached.

**Notes** — the decline is how a game kind without ranks gets no badge, exactly as with the
money readout. Every one of the ten elements is required once the panel exists.

## `SetRank`

**Contract** — fold team and rank into an index; do nothing if it is already showing; otherwise
detach whatever is attached and attach the new one.

**Notes** — the index is not range-checked. A rank at or above five, or a team index above one,
reads past the ten pictures. The callers are the game's own rank system, which is bounded by
data this file does not see.
