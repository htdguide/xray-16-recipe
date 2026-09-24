# src/xrGame/Tracer.cpp

> Draws a bullet in flight: a camera-facing streak along its path, plus a muzzle-facing disc when the round is the player's own.

**Needs** — [`Tracer.h`](Tracer.h.md) · [`xrEngine/Render.h`](../xrEngine/Render.h.md) · [`Include/xrRender/UIShader.h`](../Include/xrRender/UIShader.h.md) · [`Include/xrRender/UIRender.h`](../Include/xrRender/UIRender.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`Tracer.h`](Tracer.h.md); callers name that, not this file.
**Tier floor** — T2: it emits vertices with explicit texture coordinates into a shared draw list

## Purpose

The ballistics manager advances bullets; this draws them. It is a separate, tiny object
rather than part of the manager because the drawing needs a material and a colour table
loaded from configuration, and the manager should not own render resources.

The visible design decision is that a bullet is drawn as **two different things at once**,
and which ones appear depends on who fired it:

- a **streak** aligned with the flight direction, always drawn — the tracer proper;
- a **disc** facing the camera, drawn only for the local player's own rounds, sized by the
  round's speed.

The disc is the muzzle-flash-side of the effect seen from behind: for your own weapon the
round departs almost along the view axis, so an aligned streak degenerates into a point.
The disc is what gives your own fire something visible to read. For everyone else's fire
the streak is seen from the side and reads perfectly, and a disc would just be a blob.

## State

```text
RECORD Tracer
  material     : MaterialPass   # one pass, one texture; from configuration with defaults
  colors       : list<colour>   # indexed by the round's colour identifier
  disc_scale   : real = 2       # extra size factor for the own-fire disc
```

**Invariants** — a round's colour identifier is an index into the colour list and is
required to be in range; an out-of-range identifier is a data error in the ammunition
configuration and is caught here, where the table is.

## `CTracer` (construction)

**Contract** — reads the tracer material, its texture, the disc scale factor and the
colour table from configuration, with defaults for the first three. Builds the material
pass.

The colour table is read as a densely numbered sequence of keys under one section and
**stops at the first gap**: the loader asks for colour zero, then one, and so on, and ends
at the first missing key. So the table is defined by contiguity, not by a count, and
inserting a gap in the data silently truncates it. It is also capped at 255 entries
because the identifier is a byte.

## `Render`

**Contract** — emits the geometry for one bullet into the shared draw list. Culls the
round against the camera's frustum first, using a sphere at the streak's midpoint with a
radius of half its length, so a round entirely off screen costs one test.

```text
FUNCTION Render(head_pos, midpoint, direction, length, width, colour_id, speed, is_own_round)
  IF sphere(midpoint, length / 2) is outside the view frustum THEN RETURN
  REQUIRE colour_id is within the colour table
  colour <- colors[colour_id]

  IF is_own_round THEN
    size <- (speed / 1000) * width * disc_scale     # faster rounds read bigger: see note
    emit a camera-facing square of that size at head_pos, mapped to the disc
      region of the texture

  emit a direction-aligned quad at midpoint, half the given width and half the given
    length, mapped to the streak region of the texture
```

**Invariants**

- The streak is emitted at half the requested width and half the requested length, because
  the quad builder takes half-extents. The caller's numbers are the full visual size.
- Both sprites are two triangles with explicitly written texture coordinates, submitted as
  a triangle list into the shared draw stream rather than as indexed geometry. That is a
  concession to the draw-list interface, not a decision: a rebuild should emit quads.

**Notes** — the disc's size is proportional to the round's speed, divided by a thousand so
that a typical rifle round of roughly that muzzle velocity comes out at unit scale. The
effect is that a fast round flashes large and a slow one barely shows, which is a
reasonable stand-in for "how much of the round's flight fits in one frame". A rebuild that
derives the size from `speed × frame_time` instead gets the same look and a defensible
reason.

Both sprites take their texture regions from fixed sub-rectangles of one shared texture —
the disc from one band, the streak from another — with the coordinates written as explicit
fractions of the atlas dimensions. Those fractions are a **data contract** with the shipped
texture and must be reproduced exactly or reauthored together with it.

## The two sprite builders

**Contract** — two private routines, each emitting one camera-relevant quad:

- the **camera-facing** one spans the camera's own right and up axes, so the quad is always
  square-on to the viewer regardless of where the round is going;
- the **direction-aligned** one spans the flight direction and an axis perpendicular to
  both the flight direction and the view direction, so the quad is a ribbon along the
  path that always presents its face to the viewer.

The second is the one that makes a tracer read as a streak rather than as a flat card, and
its perpendicular is recomputed per round rather than stored, because it depends on the
viewing angle.

**Invariants** — the perpendicular is normalized defensively: a round flying exactly along
the view axis has no well-defined perpendicular, and the degenerate result must be a zero
vector (a collapsed, invisible ribbon) rather than an infinity. That case is precisely
when the viewer is looking down the round's path — which is when the disc, not the streak,
is doing the work.
