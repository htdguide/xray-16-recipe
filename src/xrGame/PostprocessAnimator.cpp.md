# src/xrGame/PostprocessAnimator.cpp

> Drives a keyframed screen-effect curve — colour grading, noise, blur, duality — as a camera effector, in four flavours that differ only in where the blend weight comes from.

**Needs** — [`PostprocessAnimator.h`](PostprocessAnimator.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`xrCore/PostProcess/PostProcess.hpp`](../xrCore/PostProcess/PostProcess.hpp.md) · [`xrEngine/EffectorPP.h`](../xrEngine/EffectorPP.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: curve evaluation and interpolation over a small parameter record

## Purpose

The engine has two halves of this idea already: a *data* half in the core (a named file of
keyframed post-process parameter curves, and the evaluation of those curves at a time) and
a *frame* half in the engine (an effector that the camera pipeline asks each frame for a
post-process parameter set, and that removes itself when it says it is finished). This
file joins them and decides the one thing neither half can decide alone: **how strongly
the curve applies right now**.

The four classes are the four answers to that question, and nothing else distinguishes
them:

| Flavour | Blend weight comes from |
|---|---|
| plain | its own time-based ease toward a target weight |
| lerp | a caller-supplied function asked every frame |
| constant lerp | a value set once by the caller |
| controlled | an owning controller object, which also decides when the effect ends |

A rebuild should express this as one effector with a pluggable weight source, which is
what the four classes already are.

## State

```text
RECORD PostprocessAnimator EXTENDS ScreenEffector, ParameterCurveSet
  cyclic       : bool          # loop forever vs play once and expire
  start_time   : real          # global clock at which the curve's time zero sat; negative = not yet started
  factor       : real          # blend weight, in [0.0001, 1]
  factor_speed : real          # ease rate toward dest_factor, or decay rate while stopping
  dest_factor  : real          # what factor eases toward while running
  stopping     : bool
  length       : real          # the curve's authored duration
  params       : ParameterSet  # the evaluated curve output for this frame
```

**Invariants**

- The blend weight never reaches zero; it is clamped to a small positive floor. That floor
  is the *sentinel for "finished"*: an effector whose weight has decayed to it reports
  itself done and is removed. A rebuild that clamps to zero instead must find another
  end-of-life signal.
- A cyclic effector has no lifetime. Its remaining lifetime is rewritten to an
  effectively-infinite value every frame rather than being set once, because the base
  effector decrements it. This is a workaround, not a design; a rebuild should give the
  base an "endless" state.
- The start time is set on the first processed frame, not at construction, so an effector
  created during a load does not appear already half-played.

## `Process`

**Contract** — asked once per frame by the camera's post-process pipeline. Advances the
blend weight, evaluates the authored curves at the current phase, resolves each parameter
against the identity (no-effect) parameter set, and blends the result into the pipeline's
accumulating parameter record by the blend weight. Returns whether the effector is still
alive; a false answer removes it.

```text
FUNCTION Process(accumulated : ParameterSet) -> bool
  IF cyclic THEN remaining_lifetime <- effectively_infinite
  base.Process(accumulated)                       # base lifetime bookkeeping

  IF start_time < 0 THEN start_time <- global_clock
  IF cyclic AND global_clock - start_time > length THEN
    start_time <- start_time + length             # advance a whole period, preserving phase
  evaluate_curves_at(global_clock - start_time)   # fills params

  IF stopping THEN factor <- factor - frame_delta * factor_speed          # linear fade out
  ELSE            factor <- factor + factor_speed * frame_delta * (dest_factor - factor)  # ease in
  factor <- clamp(factor, 0.0001, 1)

  params.color_base <- params.color_base + identity.color_base            # authored as offsets
  params.color_gray <- params.color_gray + identity.color_gray
  params.color_add  <- params.color_add  + identity.color_add

  FOR EACH channel IN (noise.intensity, noise.grain, noise.fps)
    IF the curve for that channel has no keys THEN channel <- identity's value
  IF the fps curve HAS keys THEN params.noise.fps <- params.noise.fps * 100   # authoring units

  accumulated <- lerp(identity, params, factor)
  REQUIRE accumulated.noise.grain != 0                                     # see note
  RETURN NOT (factor has reached its floor)
```

**Notes** — three details here are load-bearing and none of them is obvious.

*Phase-preserving wrap.* A cyclic effector advances its start time by exactly one period
rather than resetting it to now. Resetting would snap the curve to its start on every wrap
and drift with the frame rate; advancing keeps the effect in continuous phase across
hundreds of loops.

*Colour is authored as an offset, noise as an absolute.* The three colour curves carry
deltas from the identity grading and so are added to it; the three noise curves carry the
values themselves, and an *absent* curve means "use the identity", not "use zero". Getting
this backwards produces a black screen or a permanently grainy one.

*The noise frame rate is scaled by a hundred.* The authored number is in the data's own
units and the pipeline wants frames per second. This conversion only applies when the
channel was actually authored, which is why it sits in the else-branch of the
absent-curve test. **Could not recover**: why the authoring unit is a hundredth.

The grain check is a real guard, not a formality: grain is a divisor in the noise
computation, and a zero reaching the pipeline is a division by zero inside a shader with
no diagnostic. Failing here names the offending effect file.

## `Load`

**Contract** — reads the named curve file, optionally from the virtual filesystem rather
than from a loose path. For a non-cyclic effect, the effector's lifetime is then set to
the curve's own length, so the effect expires exactly when its animation ends. A cyclic
effect's lifetime is left alone; see the invariant above.

## `Valid`

**Contract** — whether the effector should still be kept. A cyclic effector is always
valid — it ends only by being stopped explicitly. A one-shot defers to the base's lifetime
test.

## `Stop`

**Contract** — begins the fade-out at a given rate. Only the curve half is told; the base
effector's own stop is deliberately not invoked, which means a stopped effector keeps its
base lifetime and disappears through the weight floor instead. **Could not recover**:
whether that asymmetry is intentional or a defect — the source itself marks it as
suspected.

## `CPostprocessAnimatorLerp::Process`

**Contract** — before each frame's work, replaces the blend weight with whatever the
caller's supplied weight function returns, unless the effect is fading out (in which case
the fade owns the weight). Then proceeds as the plain flavour. This is how an effect is
tied to a continuous game quantity — radiation level, health, drunkenness — rather than to
a timeline.

## `CPostprocessAnimatorLerpConst::Process`

**Contract** — the same, with a constant instead of a function: the weight is pinned to a
value set by `SetPower` until the effect is stopped.

## `CPostprocessAnimatorControlled`

**Contract** — a lerp flavour whose weight function and lifetime both come from a
controller object. Construction registers the effector with the controller and wires the
controller's weight accessor as the weight function; destruction unregisters it, so the
controller is never left pointing at a freed effector. Validity is delegated to the
controller entirely — the controller, not the timeline, decides when the effect is over.

**Invariants** — the controller and the effector hold each other, and exactly one of them
initiates the break: the effector clears the controller's reference on the way out. A
rebuild that gives the controller ownership instead must make sure the effector is removed
from the camera pipeline first.
