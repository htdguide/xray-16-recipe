# src/xrGame/UIFrameRect.cpp

> A resizable decorated rectangle: nine texture pieces — four corners, four tiled edges, one tiled interior — laid out so a frame of any size is drawn from art of one size.

**Needs** — [`UIFrameRect.h`](UIFrameRect.h.md) · [`ui/UITextureMaster.h`](../xrUICore/XML/UITextureMaster.h.md) · [`HUDManager.h`](HUDManager.h.md) · [`xrUICore/Static/UIStaticItem.h`](../xrUICore/Static/UIStaticItem.h.md)
**Used by** — reached through its declarations in [`UIFrameRect.h`](UIFrameRect.h.md); callers name that, not this file.
**Tier floor** — T2: tiling arithmetic against texture dimensions

## Purpose

Every panel, box and window border in the game's screens is one of these. The idea is the
standard nine-slice: the corners never stretch, the edges tile along their own axis, and
the middle tiles in both. What makes it worth a page is that the layout is *derived from
the art*, not authored: each piece's size is taken from its own texture region, so a
designer changes the frame's appearance by replacing the images and the geometry follows.

The second decision is that layout is **lazy**. Every setter invalidates; nothing is
recomputed until the frame is about to be drawn. A screen that moves and resizes a dozen
panels during one update therefore performs one layout each, not a dozen.

## State

```text
ENUM Part = interior | left | right | top | bottom
          | top_left | bottom_right | top_right | bottom_left

RECORD FrameRect EXTENDS SimpleWindow
  pieces      : map<Part, Sprite>    # each with its own texture region and tiling
  visible     : set<Part>            # all nine by default
  layout_valid: bool
```

**Invariants**

- The frame can never be smaller than its own corners. If a requested size would give a
  negative edge run, the size is enlarged to the corners' combined extent along that axis
  and the layout is recomputed once. Exactly once: a second recursion is a programming
  error and is asserted, because the enlarged size is by construction large enough.
- Every setter clears the valid flag. Missing one leaves a frame drawn at its previous
  geometry with no visible cause.

## `InitTextureEx` / `InitTexture`

**Contract** — binds all nine pieces from one base texture name by appending a fixed suffix
per piece: the interior, the four edges by initial, and the four corners by two initials.
The plain form uses the default interface material; the extended form takes one.

**Invariants** — the nine suffixes are a **data contract** with the shipped texture atlas
description, and so is the naming scheme itself: a frame is authored as nine named regions
sharing a prefix. A rebuild may choose another convention only by reauthoring the atlas.

## `UpdateSize`

**Contract** — computes every piece's position and tiling from the frame's current
rectangle and the pieces' own texture dimensions. Called only from the draw path, when the
layout is stale.

```text
FUNCTION UpdateSize(is_retry)
  # each piece's natural size comes from its own texture region
  sizes <- natural size of every piece

  # corners are pinned, never scaled
  top_left.position     <- origin
  top_right.position    <- origin + (width - top_right.width, 0)
  bottom_left.position  <- origin + (0, height - bottom_left.height)
  bottom_right.position <- origin + (width - .., height - ..)

  # the runs the edges must cover, between the corners
  run_top    <- width  - top_left.width  - top_right.width
  run_bottom <- width  - bottom_left.width - bottom_right.width
  run_left   <- height - top_left.height - bottom_left.height
  run_right  <- height - top_right.height - bottom_right.height

  IF any run is negative THEN
    grow the offending axis to the larger of the two corner pairs along it
    FAIL WITH "frame layout did not converge" IF is_retry
    RETURN UpdateSize(is_retry: true)

  FOR EACH edge with its run
    whole_tiles <- floor(run / tile_size)          # clamped at zero
    remainder   <- run modulo tile_size            # the partial tile at the end
    place the edge just inside its starting corner
    tile it `whole_tiles` times along its axis, plus the partial remainder

  interior: tiled in both axes the same way, placed inside the top-left corner
  layout_valid <- true
```

**Invariants**

- The remainder is carried explicitly rather than being absorbed by stretching the last
  tile. Stretching would distort a patterned border visibly at some sizes; a clipped
  partial tile is invisible.
- Whole-tile counts are floored and clamped at zero, so a run shorter than one tile draws
  only the partial piece rather than a negative count.
- The interior is tiled against the *top* and *left* runs specifically, not against the
  frame's full size, so it fills exactly the space the edges enclose.

**Notes** — the enlargement case is the only place a widget silently changes a size the
caller asked for. It is the right call — a frame drawn with overlapping corners looks
broken and the caller usually has no way to know the art's dimensions — but a rebuild
should surface it, because a layout that quietly resists a requested size is hard to debug
from the screen.

## `Draw`

**Contract** — lays the frame out if the layout is stale, then renders each of the nine
pieces that is currently marked visible, in part order. The lazy layout is performed inside
the draw, which means it happens on the render path; that is asserted rather than avoided.

The two-argument form moves the frame to a position first, if it is not already there, and
then draws — a convenience for callers that redraw the same frame at many places.

## `SetVisiblePart`

**Contract** — turns one of the nine pieces on or off. This is how a frame becomes an
open-sided panel, a divider line, or an interior with no border: the layout is unchanged,
only the drawing is suppressed. Hiding a corner does **not** reclaim the space it reserved.

## `SetWndPos` / `SetWndSize` / `SetWndRect` / `SetWidth` / `SetHeight`

**Contract** — geometry setters; each defers to the window base and then invalidates the
layout. The position setter additionally short-circuits when the new position is
indistinguishable from the old, which avoids invalidating on the many callers that
re-assert an unchanged position every frame.

## `SetTextureColor`

**Contract** — tints all nine pieces with one colour. Frames are authored in greyscale and
coloured at use, so one set of art serves every panel style in the game.

## `Update`

**Contract** — nothing. A frame has no time-dependent behaviour; the method exists because
the window base demands it.
