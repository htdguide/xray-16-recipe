# src/xrGame/pose_extrapolation.cpp

> Guessing where something will be from the last two places it was: a two-sample history at a fixed rate, and the linear fit through it.

**Needs** — [`pose_extrapolation.h`](pose_extrapolation.h.md)
**Used by** — [`pose_extrapolation.h`](pose_extrapolation.h.md)
**Tier floor** — T2: a fixed-rate sampler and a linear fit over transforms

## Purpose

A transform that changes at a coarse rate — sampled a few times a second — must be read at
the frame rate without looking stepped. Rather than interpolating *behind* the latest
sample, which costs a sample's worth of latency, this extrapolates *ahead* of it: the last
two samples define a rate of change, and any time is evaluated against that.

The tradeoff is the one every extrapolator makes. Interpolation is always right and always
late; extrapolation is always current and sometimes wrong, and is wrong in proportion to how
sharply the real motion turned since the last sample.

## State

```text
RECORD Pose
  position    : (real, real, real)
  orientation : orientation

RECORD Point
  pose : Pose
  time : real               # a sentinel far in the past until first set

RECORD Points
  points      : ring of exactly 2 Points      # oldest at index 0, newest at index 1
  last_update : int                           # clock of the last accepted sample
```

**Invariants**
- The ring holds exactly two samples and is never partially filled after initialization:
  `init` writes the same sample into both slots, so the fit is defined from the first frame
  and degenerates to "not moving".
- The two samples are in time order. The ring's push overwrites the older.
- An uninitialized pose is marked by a position at the most negative representable value and
  a zero orientation — values no real pose can take, so a misuse is visible rather than
  quietly wrong.

## Sampling

**Contract** — `update` offers a new transform and is **ignored** unless the sampling
interval has elapsed since the last accepted one. `init` forces both samples to the given
transform and restarts the clock.

```text
FUNCTION update(transform)
  IF now - last_update < sample_interval THEN RETURN
  points.push(Point{ pose_from(transform), now })
  last_update = now

FUNCTION init(transform)
  points.fill_both(Point{ pose_from(transform), now })
  last_update = now
```

**Notes** — the interval is **fifty milliseconds**, twenty samples a second. It is the
mechanism, not a throttle: sampling every frame would make the two samples one frame apart,
and a rate of change divided by a frame's worth of time is dominated by noise — the
extrapolation would jitter violently. A deliberately coarse baseline is what makes the fit
stable. Fifty milliseconds has no derivation in the source; it is the value at which the
baseline is long enough to be smooth and short enough that a turn is noticed within about
three frames.

The sample times are recorded on a real-valued clock while the interval is tested against an
integer millisecond clock. The two are the same clock at different precisions; the fit needs
the finer one.

## The pose algebra

**Contract** — four operations, each acting on position and orientation in parallel:

- `mul(v)` — scale: the position is scaled, and the orientation's *angle about its axis* is
  scaled with the axis left alone. This is what makes "half of this rotation" mean half the
  turn rather than half the numbers.
- `add(other)` — compose: positions add, orientations multiply.
- `invert` — negate the position, conjugate the orientation.
- `identity` — zero position, no rotation.

**Notes** — the algebra is **not** rigid-transform composition. A true composition would
rotate the second pose's position by the first's orientation before adding; this adds the
positions directly. The source carries the correct composition, commented out, beside the
one that ships.

The consequence is that the fit treats position and orientation as two independent linear
quantities rather than as one rigid motion. For an object that is mostly translating, or
mostly rotating about its own centre, the two agree. For an object swinging about a distant
pivot they do not, and the extrapolated position lags the arc. Given the fifty-millisecond
horizon the error is small, and the shipped code chose the cheap version knowingly. A rebuild
may use the correct composition; it will not look worse.

## The linear fit

**Contract** — from two stamped poses, produce the constant and the rate of a linear
function of time, then evaluate it. Degenerate when the two samples share a timestamp, in
which case the rate is identity — "not moving" — rather than a division by zero.

```text
FUNCTION fit(p0, p1) -> (constant, rate)
  IF p1.time - p0.time > epsilon
    rate = (inverse(p0.pose) composed with p1.pose) scaled by 1 / (p1.time - p0.time)
  ELSE
    rate = identity
  constant = inverse(rate scaled by p1.time) composed with p0.pose

FUNCTION extrapolate(out transform, time)
  p = constant composed with (rate scaled by time)
  transform = p
```

**Invariants** — the rate is the *difference* between the two poses divided by the interval,
expressed in the algebra above: invert the older, compose the newer, scale by the reciprocal
of the elapsed time. The constant is then whatever makes the line pass through the older
sample.

**Notes** — the constant is derived using `p1.time`, the *newer* sample's stamp, while being
anchored to `p0`'s pose. Substituting either sample's time into the resulting line does not
reproduce that sample's pose. This is either a transcription slip or a deliberate half-step
bias; the source gives no reason, and the effect at a fifty-millisecond baseline is a small
constant offset rather than a growing error. **Unrecovered.** A rebuild should anchor the
line at `p0` with `p0.time` and check the result against the original's motion.

The extrapolated time is on the same real-valued clock the samples carry, so a caller asks
for "the pose at this absolute moment", not "this far ahead". That is what lets the same
history serve several callers wanting different horizons.
