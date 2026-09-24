# src/xrGame/PostprocessAnimator.h

> Declares the four screen-effect effector flavours implemented in [`PostprocessAnimator.cpp`](PostprocessAnimator.cpp.md).

**Needs** — [`xrCore/PostProcess/PostProcess.hpp`](../xrCore/PostProcess/PostProcess.hpp.md) · [`xrEngine/EffectorPP.h`](../xrEngine/EffectorPP.h.md)
**Used by** — [`ActorEffector.cpp`](ActorEffector.cpp.md) · [`PostprocessAnimator.cpp`](PostprocessAnimator.cpp.md) · [`level_script.cpp`](level_script.cpp.md) · [`zone_effector.cpp`](zone_effector.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the post-process effector that plays a keyframed screen-effect curve, and its
three weight-source variants. Substance is in
[`PostprocessAnimator.cpp`](PostprocessAnimator.cpp.md).

Exported units:

- `CPostprocessAnimator` — the base: a camera post-process effector that is also a curve
  set. Constructed either with an effector slot identifier and a cyclic flag, or default
  for a subclass to configure. Overrides `Load`, `Stop`, `Valid`, `Process`.
- `CPostprocessAnimatorLerp` — takes its blend weight from a caller-supplied function,
  installed with `SetFactorFunc`.
- `CPostprocessAnimatorLerpConst` — takes its blend weight from a constant, installed with
  `SetPower`, defaulting to full strength.
- `CPostprocessAnimatorControlled` — a lerp variant bound to a controller object that
  supplies both the weight and the end-of-life decision.

## Notes

The constructor's effector-slot identifier doubles as the lifetime argument's companion: a
plain animator is created with an effectively-infinite lifetime and an "overlapped" flag,
because a one-shot's real lifetime is set from the curve's length at load time. The
declaration does not say this; [`PostprocessAnimator.cpp`](PostprocessAnimator.cpp.md)
does.
