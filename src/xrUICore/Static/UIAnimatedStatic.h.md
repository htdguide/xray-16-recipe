# src/xrUICore/Static/UIAnimatedStatic.h

> Declares the sprite-sheet animated static implemented in [`UIAnimatedStatic.cpp`](UIAnimatedStatic.cpp.md).

**Needs** — [`UIAnimatedStatic.cpp`](UIAnimatedStatic.cpp.md) · [`UIStatic.h`](UIStatic.h.md)
**Used by** — [`UIActorInfo.cpp`](../../xrGame/ui/UIActorInfo.cpp.md) · [`UIPdaWnd.cpp`](../../xrGame/ui/UIPdaWnd.cpp.md) · [`UISkinSelector.cpp`](../../xrGame/ui/UISkinSelector.cpp.md) · [`UIAnimatedStatic.cpp`](UIAnimatedStatic.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIAnimatedStatic.cpp`](UIAnimatedStatic.cpp.md). Two
decisions are stated only here.

**The animation is described by time, not by rate.** The caller gives a total duration for
the whole cycle and a frame count; the per-frame interval is derived. A rebuild that takes
frames-per-second instead must convert, because the shipped layouts state durations.

**Every parameter setter is also an invalidation.** Changing the frame count, the column
count, the duration or the cell size marks the animation's derived values stale, which
restarts it from its first frame on the next update. There is no way to change a parameter
without restarting.

## Exported units

- `CUIAnimatedStatic` — the animated picture.
- `SetFramesCount` / `SetAnimCols` / `SetFrameDimentions` / `SetAnimationDuration` /
  `SetOffset` — the grid description: how many frames, how they are arranged, how large each
  cell is in texture units, and where the grid starts within the texture.
- `Play` / `Stop` / `Rewind(offset)` — transport. The rewind offset staggers copies of one
  animation so they do not beat in unison.
- `SetCyclic` — loop, or stop at the end.
- `SetAnimPos(fraction)` — scrub to a fraction instead of playing.
- `Update` — the per-frame advance.

**Notes** — the row count of the grid has no setter and is never written; only the column
count is configurable. The implementation nonetheless divides by the row count to find the
current row, which is why non-square sheets misbehave. A rebuild should describe the grid by
columns alone and derive the rows.
