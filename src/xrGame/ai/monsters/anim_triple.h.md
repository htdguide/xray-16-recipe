# src/xrGame/ai/monsters/anim_triple.h

> Declares the prepare / execute / finalize animation component — the shape almost every creature ability is built from.

**Needs** — [`anim_triple.cpp`](anim_triple.cpp.md) · [`control_combase.h`](control_combase.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../../../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`anim_triple.cpp`](anim_triple.cpp.md) · [`bloodsucker.h`](bloodsucker/bloodsucker.h.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker/bloodsucker_vampire_execute_inline.h.md) · [`burer.h`](burer/burer.h.md) · [`burer_fast_gravi.cpp`](burer/burer_fast_gravi.cpp.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_manager_custom.h`](control_manager_custom.h.md) · [`controller.h`](controller/controller.h.md) · [`controller_state_control_hit_inline.h`](controller/controller_state_control_hit_inline.h.md) · [`zombie.cpp`](zombie/zombie.cpp.md) · [`zombie.h`](zombie/zombie.h.md)
**Tier floor** — T3: three motion handles and a three-value phase

## Purpose

Declares the surface implemented in [`anim_triple.cpp`](anim_triple.cpp.md), and fixes the
vocabulary of the three-phase animation that most creature abilities use.

## Exported units

```text
ENUM TriplePhase = { prepare, execute, finalize, none }
```

```text
RECORD TripleAnimationConfig          # what the owner fills before activating
  motions      : [prepare, execute, finalize]   # indexed by phase
  skip_prepare : bool                # start directly in execute
  execute_once : bool                # play execute once, rather than looping it
  capture      : set of { path, movement, direction }
                                     # which control channels to seize and halt
```

**Invariants** — the motion array is indexed by the phase value, so the phase enumeration's
order *is* the array layout. That is why phases advance by incrementing.

- **phase-change notification** — the event the component raises so the owning behaviour can
  act at the moment a phase begins (spawn a projectile, apply a hit, start a sound).
- **the component itself** — its capture and release, its start condition, its activation,
  its animation-end handling, and the early-exit request.
