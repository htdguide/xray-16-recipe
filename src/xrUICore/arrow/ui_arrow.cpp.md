# src/xrUICore/arrow/ui_arrow.cpp

> Turns a normalized value into a needle angle, and makes the needle chase a new value at a bounded angular speed instead of snapping to it.

**Needs** — [`ui_arrow.h`](ui_arrow.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`xrEngine/device.h`](../../xrEngine/device.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`ui_arrow.h`](ui_arrow.h.md)
**Tier floor** — T3: one interpolation over frame time and one linear map onto an angle.

## Purpose

Every analogue dial in the game — the radiation needle, the compass, the vehicle gauges — is
this widget: a rotated texture whose angle is a linear function of a value between zero and
one. The whole reason it is not just "set the rotation" is that the underlying readings are
noisy and step-wise, and a needle that teleports reads as broken instrumentation. So the
widget interposes a target and a bounded approach.

## State

```text
RECORD Arrow EXTENDS Static
  angle_begin  : real     # radians at value 0
  angle_end    : real     # radians at value 1
  angle_range  : real     # signed sweep: -(|end - begin|) clockwise, +|end - begin| otherwise
  ang_velocity : real     # value units per second the needle may travel
  pos          : real     # displayed value, 0..1
  temp_pos     : real     # the value being approached, 0..1
```

**Invariants** — both `pos` and `temp_pos` are clamped into the unit interval at every write;
the widget's rotation is always exactly `angle_begin + pos * angle_range`, so nothing else may
write the rotation.

## `init_from_xml`

**Contract** — attaches the needle to a parent, marks it owned by the tree, reads the ordinary
static appearance from the same element, then reads four dial attributes: start angle, end
angle, angular speed and a clockwise flag. Direction is folded into the sweep's sign at load
time so the per-frame path does no branching on it.

**Notes** — the defaults are a full turn starting at zero, speed one, clockwise. A dial that
only sweeps part of a circle states both angles; the two are read independently rather than as
a start plus a span, because the shipped layouts are authored that way.

## `SetNewValue`

**Contract** — advances the needle one frame toward `new_value`. Call it every frame with the
current reading; it is not a "start an animation" call. Returns nothing and never blocks.

```text
FUNCTION set_new_value(new_value)
  clamp new_value into 0..1

  IF pos is indistinguishable from temp_pos
    # settled: accept a new target, slightly overshooting it
    temp_pos <- pos + 1.05 * (new_value - pos)
    clamp temp_pos into 0..1
  ELSE
    # in flight: ignore the new reading entirely this frame and
    # travel toward the target we already have
    remaining <- temp_pos - pos
    step      <- ang_velocity * frame_seconds
    step      <- min(|step|, |remaining|) with the sign of remaining
    pos       <- pos + step

  clamp pos into 0..1
  set_pos(pos)
```

**Notes** — two decisions here look arbitrary and are not.

The overshoot factor of 1.05 means the needle aims slightly past the reading. Because a new
target is only accepted once the needle has *settled*, and settling is decided by an
approximate comparison, aiming exactly at the reading could leave the needle parked a hair
short of the target and unable to accept the next one. Overshooting guarantees the approach
crosses the target and the comparison closes. It also gives a real instrument's slight
overrun.

While a movement is in flight the incoming reading is discarded rather than retargeted. The
needle therefore always completes a swing before it notices the world changed, which is what
makes a jittering reading look like a smoothly lagging needle rather than a vibrating one.
It also means a value that changes faster than the needle travels is never fully tracked — a
deliberate low-pass, not an oversight.

The step is scaled by the frame's elapsed seconds, so the speed is in value units per second
and the needle behaves the same at any frame rate.

## `SetPos`

**Contract** — writes the value and the rotation together, with no chasing. This is the only
place the rotation is computed, and it is what makes the chase and the immediate set produce
identical geometry.
