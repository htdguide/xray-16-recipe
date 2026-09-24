# src/xrGame/ui/UISleepStatic.cpp

> Paints the span of hours a nap will cover, by cutting a window out of a 24-hour-wide strip texture
> and wrapping it around midnight.

**Needs** — [`UISleepStatic.h`](UISleepStatic.h.md) · [`../Actor_Flags.h`](../Actor_Flags.h.md) · [`../Level.h`](../Level.h.md) · [`../date_time.h`](../date_time.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md)
**Used by** — [`UISleepStatic.h`](UISleepStatic.h.md)
**Tier floor** — T1: computes texture-space pixel rectangles against a texture's known dimensions

## Purpose

The sleep screen shows the player which part of the day their nap will cover. The artwork is a
single strip texture laid out as 24 hours across 2048 pixels, and this widget shows the sub-range
the nap occupies. Because a nap can cross midnight, the range can wrap — so the widget draws **two**
pieces, a main one and a remainder, and the remainder is usually degenerate.

## State

```text
RECORD SleepStrip extends Picture
  piece_a : Quad      # the main window into the strip, always drawn
  piece_b : Quad      # the wrapped remainder; sized 1x1 when there is no wrap
```

Invariants:

- Both pieces are bound to the same texture and material; binding one without the other leaves the
  wrapped part invisible.
- `piece_b` is never hidden, only shrunk to a single unit — the widget has no visibility flag per
  piece, so "no wrap" is expressed as a degenerate size.
- `piece_b` is always positioned immediately to the right of `piece_a`, so together they read as one
  continuous span.

## The strip geometry

The three numbers are frozen by the shipped artwork:

- the strip is **2048 pixels wide** and **128 pixels tall**;
- one hour is **85 pixels** — note that 24 × 85 = 2040, eight pixels short of 2048, so the last
  eight pixels of the strip are never addressed;
- a nap is **7 hours** wide on the strip.

The eight-pixel shortfall is not explained anywhere in the source. It is consistent with the artwork
having been authored as 24 columns of 85 pixels inside a power-of-two texture, with the remainder
left blank.

## `Update`

**Contract** — Recomputes both pieces from the current game clock and the configured sleep start
offset. Runs every frame. Does not allocate.

```text
FUNCTION Update()
  hour <- hour-of-day from the current game time
  hour <- (hour + sleep_start_offset - 1) MOD 24      # the hour the nap begins

  start <- hour * 85
  end   <- (hour + 7) * 85
  wrap  <- 0
  IF end > 2048 THEN
    wrap <- end - 2048        # the part past the end of the strip
    end  <- 2048

  origin <- this widget's absolute position (own position plus the parent's)

  piece_a.texture_rect <- (start, 0) .. (end, 128)
  piece_a.size         <- ((end - start) scaled by the horizontal canvas factor, 128)
  piece_a.position     <- origin

  IF wrap > 0 THEN
    piece_b.texture_rect <- (0, 0) .. (wrap, 128)
    piece_b.size         <- (wrap scaled by the horizontal canvas factor, 128)
    piece_b.position     <- immediately right of piece_a
  ELSE
    piece_b.size         <- (1, 1)                    # degenerate: effectively invisible
```

**Notes** — The `- 1` in the hour is authored: the configured sleep duration setting is read as an
*inclusive* hour count, so a setting of 1 means the nap begins in the current hour.

Widths are multiplied by the horizontal canvas-to-screen factor but heights are not, and the height
is used raw at 128. That is because the strip is drawn at texture scale horizontally — a pixel of
strip per pixel of screen — while its height is a fixed canvas height. It is a deliberate mismatch
and copying it is what reproduces the original's appearance on a wide display; see chapter 15's note
that the canvas scale is non-uniform.

The origin is computed as own position plus the *immediate parent's* position rather than as a full
absolute rectangle. That is correct only while this widget is exactly one level below a
screen-rooted window, which is how the shipped sleep screen places it, and is a latent constraint a
rebuild should either preserve or replace with a real absolute-rectangle query.

## `Draw`

**Contract** — Draws the two pieces and **nothing else** — it deliberately does not call the base
picture's draw, so the widget's own texture, text and children are not painted. The pieces are the
entire visual.

## `InitTextureEx`

**Contract** — Binds the named texture to the base picture (which is what resolves the material
name), then binds the *same* texture and resolved material to the second piece, and parks the second
piece at the widget's position at a degenerate size until `Update` runs. Both pieces must be bound
here; there is no other binding point.
