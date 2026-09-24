# src/xrGame/ai/monsters/pseudogigant/pseudo_gigant_step_effector.cpp

> One footfall, felt: three out-of-phase oscillations applied to the camera's orientation, front-loaded so the jolt arrives at the moment of impact and dies away.

**Needs** — [`pseudo_gigant_step_effector.h`](pseudo_gigant_step_effector.h.md) · [Seam: Graphics device](../../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`pseudo_gigant_step_effector.h`](pseudo_gigant_step_effector.h.md)
**Tier floor** — T1: rebuilds an orientation basis and composes rotations per frame inside the camera pipeline

## Purpose

The camera effect that makes the giant weigh something. Two decisions carry it: the envelope
(a sharp attack that decays, not a symmetric fade) and the three-axis shape (three sinusoids at
different frequencies, amplitudes and phases, so the shake reads as a ground impact rather than
as a wobble).

## `ProcessCam`

**Contract** — called once per frame by the camera pipeline with the camera's current position,
forward direction and up direction. Consumes the frame's elapsed time, and either rewrites the
two direction vectors in place and reports that it is still alive, or reports that it is
finished and leaves them untouched. Position is never modified — the effect is purely
rotational, so it cannot push the camera into geometry.

```text
FUNCTION ProcessCam(camera) -> bool
  remaining = remaining - frame_elapsed
  IF remaining < 0
    RETURN false                      # finished; the stack drops it

  t = remaining / total               # 1 at the start, falling to 0
  elapsed_fraction = 1 - t

  # rebuild an orthonormal basis from the camera's own up and forward
  basis.up      = camera.up
  basis.forward = camera.forward
  basis.right   = cross(camera.up, camera.forward)

  cycle = periods * TWO_PI

  # the envelope: inverse-square in a term that starts small and grows
  k   = elapsed_fraction + epsilon + (1 - power)
  amp = max_amp * (PI / 180) / (10 * k * k)

  angles.heading = amp / 2 * sin(cycle     * elapsed_fraction)
  angles.pitch   = amp     * cos(cycle / 2 * elapsed_fraction)
  angles.roll    = amp / 4 * sin(cycle / 4 * elapsed_fraction)

  rotated = compose(basis, rotation_from(angles))
  camera.forward = rotated.forward
  camera.up      = rotated.up
  RETURN true
```

**Invariants**

- **The envelope is an inverse square, so the shake is strongest at the instant of impact and
  falls off fast** — that is what makes it read as a blow rather than a tremor. The denominator
  term starts at `1 - power` and grows to `2 - power`, so a *nearby* footfall (power near its
  maximum) starts with a small denominator and therefore a large amplitude, and a distant one
  starts with a denominator near one and is muted from the outset. The falloff is thus applied
  twice — once to the amplitude at construction and once to the envelope's shape here — which
  makes the distance dependence markedly steeper than linear. That is a choice, not a slip, but
  nothing in the source records why the curve was wanted rather than a single scaling.
- **The epsilon term exists to stop division by zero** on the first frame, where the elapsed
  fraction and, at maximum power, the whole denominator would be zero.
- **The three axes differ in every respect.** Pitch carries the full amplitude and half the
  frequency and starts at its extreme (a cosine), which is the vertical jolt. Heading carries
  half the amplitude at the full frequency, and roll a quarter at a quarter frequency; both
  start at zero (sines). The result never repeats within a footfall and never reads as a
  regular wobble.
- **Amplitude is authored in degrees and converted here.** The configuration is written in the
  units a designer thinks in.
- **The basis is rebuilt from the camera's live vectors every frame**, so the shake composes
  correctly with whatever else is steering the camera — including a second footfall's effect
  already in the stack.

**Notes** — the divisor of ten in the amplitude, the halving and quartering of the three axes,
and their frequency ratios are all compiled in with no derivation anywhere. They are the
effect's signature and a rebuild that changes them changes how a giant feels, but there is no
recoverable reason for these particular values over neighbouring ones.
