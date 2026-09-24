# src/xrGame/debug_renderer.cpp

> Turns the three volume shapes the game layer wants to visualize — oriented box, axis-aligned box, ellipsoid — into indexed line lists for the renderer's debug channel.

**Needs** — [`debug_renderer.h`](debug_renderer.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: geometry generation into a vertex/index pair

## Purpose

Debugging this engine means looking at volumes: a restrictor's box, a zone's sphere, a
creature's bounding box, a vision frustum. This file generates the wireframes for them. It
holds no state and owns no device resources; everything is handed straight down to the
renderer's debug channel, which batches across the frame and draws once.

The file exists only in debug builds.

## State

Stateless.

## `add_lines`

**Contract** — the single funnel. Takes a vertex array, a count, an array of index *pairs*,
a pair count and one colour for the whole batch, and appends them to the renderer's queue.
Every shape below reduces to one call. The line-list form — vertices plus index pairs
rather than a flat vertex stream — matters because the shapes reuse vertices heavily: the
ellipsoid draws over a thousand segments from a hundred and fourteen points.

## `draw_obb(matrix, color)`

**Contract** — draws the unit cube spanning minus one to plus one on each axis, transformed
by the matrix. The matrix therefore carries the box's scale as well as its orientation and
position, which is why this overload takes no size. Twelve edges from eight corners.

**Notes** — the corner ordering is fixed and the twelve index pairs are written out
literally: four edges on the near face, four on the far, four connecting them. A rebuild can
generate them; nothing depends on the particular ordering.

## `draw_obb(matrix, half_size, color)`

**Contract** — the convenience form: builds a scale from the half-extents, composes it
*before* the given matrix, and delegates. Composition order is load-bearing — the box is
scaled in its own local space and then oriented and placed, not scaled along world axes.

## `draw_ellipse`

**Contract** — draws a unit sphere through the matrix, so an arbitrary affine matrix yields
an arbitrary ellipsoid. The sphere is a **baked table** of 114 points and 1 037 index pairs,
not generated: seven latitude rings of sixteen points each, at the eighth-turn cosines
(±0.9239, ±0.7071, ±0.3827 and zero), plus the two poles. The index pairs join each point to
its ring neighbours, to its neighbours on the adjacent rings, and to the poles.

**Invariants** — the table is a *unit* sphere; every visual property except shape comes from
the matrix. The point positions are the sines and cosines of multiples of an eighth of a
turn, so a rebuild can compute them rather than transcribing them, and should — the table is
four hundred lines of literals whose only content is that the sphere has sixteen segments
around and eight from pole to pole.

**Notes** — the transform is applied to a *local copy* of the table in place, so the table
itself is not corrupted across calls. That is the only reason the table can be a constant
and a rebuild should keep it read-only and transform into a scratch buffer.
