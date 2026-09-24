# src/xrGame/debug_renderer_inline.h

> The two cheapest debug shapes and the per-frame flush.

**Needs** — [`debug_renderer.h`](debug_renderer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: two-vertex geometry

## Purpose

The small half of [`debug_renderer.cpp`](debug_renderer.cpp.md): the single-segment draw,
the axis-aligned box, and the flush. Split out for inlining, which is why the line draw —
by far the most-called of the family — lives here rather than with the rest.

## State

Adds nothing.

## `draw_line`

**Contract** — queues one segment between two points, transformed by the matrix, in one
colour. Builds a two-vertex array and a single index pair and hands them to the funnel.
Callers that want world space pass the identity.

**Notes** — every call allocates its own two-element batch. For a debug channel drawing a
few hundred lines a frame that is irrelevant; a rebuild drawing thousands should let the
caller append into a shared buffer instead.

## `draw_aabb`

**Contract** — an axis-aligned box from a centre and three separate half-extents. Builds a
pure translation matrix and delegates to the oriented-box draw with the half-extents as the
size — the box is axis-aligned precisely because the matrix carries no rotation.

## `render`

**Contract** — flushes everything queued this frame to the renderer's debug channel. Called
once per frame by the level. Wrapped in a profiler zone, since a debug build with every
overlay on can spend a measurable share of the frame here.
