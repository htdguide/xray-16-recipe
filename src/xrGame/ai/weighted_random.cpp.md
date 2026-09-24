# src/xrGame/ai/weighted_random.cpp

> Draws from a one-, two- or three-point piecewise-linear distribution by rejection sampling inside a trapezoid. Never called.

**Needs** — [`weighted_random.h`](weighted_random.h.md)
**Used by** — [`weighted_random.h`](weighted_random.h.md)
**Tier floor** — T3: arithmetic and two uniform draws

## Purpose

Implements the draw. The algorithm is worth recording even though the file is dead, because
a rebuild wanting authorable AI timing distributions will want exactly this and will
otherwise reinvent it badly.

The construction: the points are read as vertices of a density that is linear between
consecutive points and zero outside. Drawing then has two levels.

**Three points** — choose which of the two segments to draw from, in proportion to each
segment's area, then recurse into that segment as a two-point draw. Each segment's area is
computed as (sum of the endpoint weights) times (the span between them), which is twice the
true trapezoid area — but since only the *ratio* of the two areas is used, the factor
cancels. When the two areas together are effectively zero, the distribution has collapsed
and the first value is returned.

**Two points** — rejection sampling in the bounding rectangle. Draw a value uniformly along
the span and a weight uniformly up to the sum of the endpoint weights; if the drawn weight
is under the density's height at the drawn value, accept it, otherwise *reflect* the value
about the segment's midpoint and accept that. Reflection rather than redrawing is what makes
the draw terminate in constant time: the rejected half of the rectangle maps exactly onto
the accepted half of the complementary triangle.

**One point** — return it.

## `generate`

**Contract** — returns one sample. Does not allocate. Not thread-safe, and not
deterministic against the engine's own seed — see the Notes, which is the finding that
matters here.

```text
FUNCTION generate() -> real
  IF b and c are both live                            # three points
    area_ab = (a_weight + b_weight) * (b_value - a_value)
    area_bc = (b_weight + c_weight) * (c_value - b_value)
    IF area_ab + area_bc < 0.0001 THEN RETURN a_value  # degenerate: no spread
    IF uniform01() < area_ab / (area_ab + area_bc)
      RETURN two_point_draw(a, b)
    ELSE
      RETURN two_point_draw(b, c)

  ELSE IF b is live                                    # two points
    u1 = uniform01()
    u2 = uniform01()
    IF |a_weight - b_weight| < 0.0001
      RETURN a_value + (b_value - a_value) * u1        # flat: a plain uniform draw
    drawn_value  = a_value + (b_value - a_value) * u2
    drawn_weight = (a_weight + b_weight) * u1
    height_here  = a_weight + (drawn_value - a_value) * slope   # linear interpolation
    IF drawn_weight < height_here
      RETURN drawn_value
    RETURN a_value + b_value - drawn_value             # reflect about the midpoint

  ELSE
    RETURN a_value                                      # one point: a constant
```

**Invariants** — the reflection trick requires the density to be *monotone* across the
segment, which two endpoints guarantee. It would not survive a segment with an interior
vertex, which is exactly why the three-point case splits into two-point draws rather than
sampling the whole shape at once.

## Notes

**It draws from the C library's global generator, not the engine's.** Every other random
draw in the AI layer goes through the engine's own seeded generator, which is what makes
physics and AI reproducible run to run on one machine — conformance criterion 8. This file
does not. Had it ever been called, it would have been a determinism hole. A rebuild must
route it through the engine's generator.

**The slope is computed with the endpoints in opposite orders** in the two branches (rising
versus falling), which is arithmetically correct but is written as two near-identical blocks
differing in one subtraction. A rebuild should write it once.

**`is_const` is declared and never called**, like the rest of the type.
