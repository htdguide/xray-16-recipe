# src/xrGame/ai/monsters/pseudogigant/pseudo_gigant_step_effector.h

> Declares the camera shake a pseudogiant's footfall pushes onto the player's view.

**Needs** — [`pseudo_gigant_step_effector.cpp`](pseudo_gigant_step_effector.cpp.md) · [`../../../CameraEffector.h`](../../../CameraEffector.h.md)
**Used by** — [`pseudo_gigant.cpp`](pseudo_gigant.cpp.md) · [`pseudo_gigant_step_effector.cpp`](pseudo_gigant_step_effector.cpp.md)
**Tier floor** — T2: a camera effect with a lifetime, evaluated per frame

## Purpose

Declares the surface implemented in
[`pseudo_gigant_step_effector.cpp`](pseudo_gigant_step_effector.cpp.md). One effect in the
camera's effect stack, identified by its own tag so the stack can recognise and replace it.

## State

```text
RECORD StepShake
  total     : real     # the full duration, kept because the remaining time is normalised against it
  max_amp   : real     # authored amplitude already multiplied by the distance falloff
  periods   : real     # how many oscillations fit into the duration
  power     : real     # the distance falloff, 0 at the rim to about 0.83 underfoot
  # plus the base effect's remaining lifetime
```

**Invariants** — the amplitude is scaled by the falloff *at construction* and the falloff is
*also* kept, because it is used a second time in the decay curve. Scaling twice is deliberate,
not a mistake; see the implementation.

## Exported units

- construction — takes duration, amplitude, period count and the distance falloff.
- `ProcessCam` — one frame of the effect; returns whether it is still alive.
