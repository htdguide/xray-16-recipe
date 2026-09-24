# src/xrCore/Animation/interp.cpp

> Evaluates an animation curve at a time: the six key shapes, the six out-of-range behaviours, and the tangent rules that make them agree.

**Needs** — [`Envelope.hpp`](Envelope.hpp.md)
**Used by** — [`Envelope.cpp`](Envelope.cpp.md) · [`Envelope.hpp`](Envelope.hpp.md)
**Tier floor** — T2: floating-point curve arithmetic.

## Purpose

The whole of curve evaluation. This is a faithful adoption of a 1990s animation package's published envelope evaluator, and **its exact behaviour is the compatibility surface**: every weather curve, every hand-authored bone channel and every post-process ramp in the shipped data was authored against this evaluator, and a "better" spline produces visibly different motion.

The thing worth understanding before reading it is that a key's shape describes the segment **ending** at that key, not starting from it, and that the tangent at a key depends on its *neighbours* — so evaluating one segment needs up to four keys.

## `evalEnvelope`

**Contract** — given a curve and a time, return the interpolated value. An empty curve is zero. A single-key curve is that key's value everywhere. A time outside the keyed range is handled by the corresponding end behaviour, which may return directly or may rewrite the time (and accumulate a value offset) and fall through to normal interpolation.

```text
FUNCTION evaluate(e, time) -> real
  IF e.keys IS EMPTY THEN RETURN 0
  IF count(e.keys) = 1 THEN RETURN e.keys[0].value
  first := e.keys[0];  last := e.keys[count-1]
  offset := 0

  IF time < first.time THEN
    SELECT e.behaviour[0]
      CASE reset:     RETURN 0
      CASE constant:  RETURN first.value
      CASE repeat:    time := wrap(time, first.time, last.time)
      CASE oscillate: time, cycles := wrap_counted(time, first.time, last.time)
                      IF cycles IS ODD THEN
                        time := last.time - first.time - time
      CASE offset:    time, cycles := wrap_counted(time, first.time, last.time)
                      offset := cycles * (last.value - first.value)
      CASE linear:    slope := outgoing_tangent(none, first, e.keys[1])
                               / (e.keys[1].time - first.time)
                      RETURN slope * (time - first.time) + first.value
  ELSE IF time > last.time THEN
    ... the mirror image, using behaviour[1] and the incoming tangent at `last`

  # locate the segment
  k := the largest index WITH e.keys[k+1].time >= time
  key0 := e.keys[k];  key1 := e.keys[k+1]
  key0_prev := e.keys[k-1] IF k > 0 ELSE none
  key1_next := e.keys[k+2] IF k+2 < count ELSE none

  IF time = key0.time THEN RETURN key0.value + offset      # exact hits first:
  IF time = key1.time THEN RETURN key1.value + offset      # avoids a 0/0 below

  t := (time - key0.time) / (key1.time - key0.time)        # normalized to [0,1]

  SELECT key1.shape                          # the SEGMENT's shape is key1's
    CASE tcb, bezier, hermite:
      out := outgoing_tangent(key0_prev, key0, key1)
      in  := incoming_tangent(key0, key1, key1_next)
      h1, h2, h3, h4 := hermite_basis(t)
      RETURN h1*key0.value + h2*key1.value + h3*out + h4*in + offset
    CASE bezier2:  RETURN bezier2(key0, key1, time) + offset
    CASE linear:   RETURN key0.value + t*(key1.value - key0.value) + offset
    CASE stepped:  RETURN key0.value + offset
    OTHERWISE:     RETURN offset
```

**Invariants** — the exact-hit tests must come before the normalization: a zero-length segment would otherwise divide by zero. They must also come *after* the offset is computed, so a wrapped time landing exactly on a key still carries its accumulated offset.

The segment search walks from the start on every call. For a curve with a handful of keys, which is almost all of them after the constant-channel simplification, that is cheaper than any index. A rebuild evaluating long curves should add a cursor.

`wrap_counted` reports how many whole ranges were crossed *and* in which direction, which is what makes `oscillate` and `offset` work in both directions from the keyed range.

## `outgoing` / `incoming` — the tangent rules

**Contract** — the tangent leaving `key0` and the tangent arriving at `key1`, each computed from the key's own shape and scaled by the ratio of adjacent interval lengths so that unevenly spaced keys do not overshoot.

```text
FUNCTION outgoing_tangent(prev, key0, key1) -> real
  SELECT key0.shape
    CASE tcb:
      a := (1 - key0.tension) * (1 + key0.continuity) * (1 + key0.bias)
      b := (1 - key0.tension) * (1 - key0.continuity) * (1 - key0.bias)
      d := key1.value - key0.value
      IF prev EXISTS THEN
        ratio := (key1.time - key0.time) / (key1.time - prev.time)
        RETURN ratio * (a * (key0.value - prev.value) + b * d)
      ELSE RETURN b * d
    CASE linear:
      d := key1.value - key0.value
      IF prev EXISTS THEN
        ratio := (key1.time - key0.time) / (key1.time - prev.time)
        RETURN ratio * (key0.value - prev.value + d)
      ELSE RETURN d
    CASE bezier, hermite:
      out := key0.param[1]                    # the stored outgoing tangent
      IF prev EXISTS THEN out := out * ratio
      RETURN out
    CASE bezier2:
      # handles are (time, value) offsets; the tangent is their slope,
      # rescaled to this segment's length
      out := key0.param[3] * (key1.time - key0.time)
      IF |key0.param[2]| > 1e-5 THEN RETURN out / key0.param[2]
      ELSE                           RETURN out * 1e5        # near-vertical
    CASE stepped, OTHERWISE: RETURN 0
```

`incoming_tangent` is the mirror: it reads `key1`'s shape, uses `param[0]` as the stored tangent, `param[0..1]` as the handle offsets, and swaps which of the two tension-continuity-bias coefficients multiplies which difference.

**Invariants** — the coefficient swap between the two directions is what *continuity* means: a non-zero continuity makes the incoming and outgoing tangents differ, producing a corner. Using the same coefficients in both directions removes the parameter's only effect.

**Notes** — the near-vertical guard replaces a division by a tiny handle-time with a multiplication by a large constant, which bounds the tangent instead of producing an infinity. The two constants (the threshold and the substitute slope) are reciprocals, so the substitution is continuous with the division at the threshold. That is the reason they are what they are.

The interval-ratio scaling is the classic fix for tension-continuity-bias splines on unevenly spaced keys, and dropping it makes fast-then-slow animation overshoot on the fast side.

## `bez2` / `bez2_time` — the two-dimensional bezier

**Contract** — a two-dimensional bezier key treats *time* as a coordinate as well as a parameter, so evaluating it at a given time first requires solving for the curve parameter that produces that time, and then evaluating the value curve at that parameter.

```text
FUNCTION bezier2(key0, key1, time) -> real
  # control points on the TIME axis
  x0 := key0.time
  x1 := IF key0 IS bezier2 THEN key0.time + key0.param[2]
        ELSE key0.time + (key1.time - key0.time) / 3      # default handle
  x2 := key1.time + key1.param[0]
  x3 := key1.time
  t  := solve_for_parameter(x0, x1, x2, x3, target: time)

  # control points on the VALUE axis
  y0 := key0.value
  y1 := IF key0 IS bezier2 THEN key0.value + key0.param[3]
        ELSE key0.value + key0.param[1] / 3
  y2 := key1.value + key1.param[1]
  y3 := key1.value
  RETURN cubic_bezier(y0, y1, y2, y3, t)

FUNCTION solve_for_parameter(x0, x1, x2, x3, target) -> real
  # bisection on [0, 1]; the time curve is monotone for well-formed handles
  lo := 0; hi := 1
  REPEAT
    t := (lo + hi) / 2
    v := cubic_bezier(x0, x1, x2, x3, t)
    IF |target - v| <= 0.0001 THEN RETURN t
    IF v > target THEN hi := t ELSE lo := t
```

**Invariants** — the tolerance is absolute, in the curve's *time* units, which are seconds. A curve spanning thousands of seconds converges to a coarser parameter than one spanning a fraction of a second. Nothing in the shipped data spans long enough for that to matter.

The default handles when the previous key is *not* a two-dimensional bezier place the outgoing control point one third of the way along the segment — the standard construction that makes a bezier segment agree with a linear one when the handles are flat.

**Notes** — the solver is written recursively with no depth limit. A non-monotone time curve, which a user can author by dragging a handle backwards past the previous key, never converges. A rebuild should bound the iteration count; twenty bisections exhaust a single-precision real's resolution.

## `hermite`

**Contract** — the four cubic Hermite basis weights at a normalized parameter: two for the endpoint values and two for the endpoint tangents.

```text
FUNCTION hermite_basis(t) -> (h1, h2, h3, h4)
  t2 := t*t;  t3 := t*t2
  h2 := 3*t2 - 2*t3        # weight of the SECOND value
  h1 := 1 - h2             # weight of the first value
  h4 := t3 - t2            # weight of the incoming tangent
  h3 := h4 - t2 + t        # weight of the outgoing tangent
```

**Notes** — writing the first weight as one minus the second guarantees they sum to exactly one in floating point, so a segment between two equal values is exactly that value everywhere. Computing both independently lets rounding leak in, which shows up as a bone that drifts by a fraction of a millimetre through a hold. That is the kind of detail worth copying.

## `range`

**Contract** — fold a value into a half-open interval and report how many interval-widths were crossed, with a sign. A zero-width interval returns the lower bound and zero crossings.

```text
FUNCTION wrap_counted(v, lo, hi) -> (wrapped, cycles)
  width := hi - lo
  IF width = 0 THEN RETURN (lo, 0)
  wrapped := lo + v - width * floor(v / width)
  cycles  := -round_away_from_zero((wrapped - v) / width)
  RETURN (wrapped, cycles)
```

**Notes** — the cycle count is derived from the *difference* rather than from the same division that produced the wrap, and it is rounded away from zero by adding a signed half before truncating. Deriving it directly would disagree with the wrap by one at the interval boundaries, which is exactly where an oscillating curve flips direction.
