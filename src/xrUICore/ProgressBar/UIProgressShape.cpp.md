# src/xrUICore/ProgressBar/UIProgressShape.cpp

> Draws progress as a ring of textured triangular sectors fanned from a centre, each sector's opacity set by a smooth step across the progress fraction.

**Needs** — [`UIProgressShape.h`](UIProgressShape.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`ui_base.h`](../ui_base.h.md) · [`Include/xrRender/UIRender.h`](../../Include/xrRender/UIRender.h.md) · [`Include/xrRender/UIShader.h`](../../Include/xrRender/UIShader.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIProgressShape.h`](UIProgressShape.h.md)
**Tier floor** — T1: it builds a triangle list with explicit positions, texture coordinates and per-vertex colours and submits it to the device directly, rather than going through a widget.

## Purpose

The circular progress indicator — a clock face, a radiation meter, a loading ring. It is the
only widget in the toolkit that emits its own geometry: everything else draws axis-aligned
quads through a static. That is why it lives at a lower tier than its neighbours.

The idea is a triangle fan: the ring is divided into a fixed number of sectors, each drawn as
a triangle from the centre out to two points on the circle, textured by the *same* mapping —
so the texture is sampled as if the fan were a disc cut out of it — and tinted by an opacity
that depends on how far the sector is along the progress.

## State

```text
RECORD ProgressShape EXTENDS Static
  stage        : real          # progress, 0..1
  sector_count : int           # 8 unless set
  clockwise    : bool
  blend        : bool          # smooth the sector opacities, or make them binary
  angle_begin  : real          # radians; 0
  angle_end    : real          # radians; a full turn
  texture      : optional<Static>   # the ring art; self when absent
  background   : optional<Static>   # drawn underneath
  show_text    : bool
```

**Invariants** — the "origin" widget is the texture child when one exists and the shape itself
otherwise; every geometric quantity — rectangle, texture rectangle, shader — comes from that
one widget, so the two cases never mix.

## `set_pos`

**Contract** — two forms. The fractional form stores the progress directly. The integer form
stores `pos / max` and, when text is enabled, additionally writes `pos` as the origin's text —
so the ring shows a count while the fan shows the fraction.

## `draw`

**Contract** — draws the backdrop if present, then the text if enabled, then builds and submits
one triangle list of `3 × sector_count` vertices.

```text
FUNCTION draw()
  IF background EXISTS THEN background.draw()
  origin <- texture OR self
  IF show_text THEN origin.draw_text()

  device.set_shader(origin.shader)
  atlas <- origin.shader.base_texture_resolution
  device.begin_primitive(sector_count * 3, triangle_list)

  # two coincident coordinate systems: one in screen pixels, one in
  # normalized texture space, both centred and both of the same angular shape
  screen_rect  <- origin.absolute_rect converted to pixels
  centre_pos   <- centre(screen_rect)      ; radius_pos <- width(screen_rect) / 2
  tex_rect     <- origin.texture_rect / atlas
  centre_tex   <- centre(tex_rect)         ; radius_tex <- width(tex_rect) / 2

  sweep <- (angle_end - angle_begin) magnitude, negated when clockwise
  angle <- angle_begin
  prev_pos <- rotate((0, -radius_pos), angle)
  prev_tex <- rotate((0, -radius_tex), angle)

  FOR i IN 0 .. sector_count - 1
    alpha <- sector_opacity(i + 1, sector_count, stage, blend)
    colour <- white at that alpha

    emit vertex at centre_pos  with texture coordinate centre_tex
    a_pos <- centre_pos + prev_pos ; a_tex <- centre_tex + prev_tex
    angle <- angle + sweep / sector_count
    prev_pos <- rotate((0, -radius_pos), angle)
    prev_tex <- rotate((0, -radius_tex), angle)
    b_pos <- centre_pos + prev_pos ; b_tex <- centre_tex + prev_tex

    emit a_pos, b_pos in the winding order the direction demands
  device.flush_primitive()
```

**Notes** — the radius comes from the rectangle's *width* in both spaces, so a non-square
widget or a non-square texture rectangle produces a circle inscribed in the width, not an
ellipse filling the box. That is a limitation, not a decision, but it is observable and the
shipped art is square.

The sweep's sign is the only thing the clockwise flag changes about the geometry; it also flips
the two outer vertices' order so the triangles keep a consistent winding regardless of
direction. A rebuild whose rasterizer does not cull these triangles can drop the second half.

The shape is drawn in the same pass as every other UI quad and therefore in tree order; it does
not defer.

## `sector_opacity`

**Contract** — maps a sector's ordinal onto an opacity. In blended mode it is a logistic curve
centred on the progress boundary, so the two or three sectors nearest the current position fade
rather than switching; in unblended mode it is a hard step — full opacity for every sector
before the boundary, none after.

```text
FUNCTION sector_opacity(index, total, stage, blend) -> real
  boundary <- stage * (total + 1)
  IF blend
    RETURN 1 / (exp((index - boundary) * 0.9) + 1)
  RETURN IF index < boundary THEN 1 ELSE 0
```

**Notes** — the curve's steepness, 0.9, and the `total + 1` in the boundary are both tuned
constants with no derivation in the source. Their joint effect is that the ring reaches full
opacity slightly before the progress reaches 1 and that the fade spans roughly two sectors at
the default count of eight. Recorded as unrecovered; they must be copied to match the shipped
look.
