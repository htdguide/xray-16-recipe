# src/xrGame/SleepEffector.cpp

> The screen effect that covers falling asleep and waking up: fade the picture into a sleeping state, hold it for as long as sleep lasts, then fade back.

**Needs** — [`SleepEffector.h`](SleepEffector.h.md) · [`xrEngine/EffectorPP.h`](../xrEngine/EffectorPP.h.md)
**Used by** — reached through its declarations in [`SleepEffector.h`](SleepEffector.h.md); callers name that, not this file.
**Tier floor** — T3: an interpolation against a lifetime

## Purpose

Sleeping is a game action of *unknown duration*: the player chooses to sleep some hours,
the screen fades, the world advances, and the screen fades back. A screen effector,
however, is built around a known lifetime that counts down to expiry. Reconciling the two
is the whole content of this file, and the trick is small and worth stating plainly: the
effector **stops its own clock** while sleep is in progress, and resumes it when the game
says the sleeping is over.

Everything else — which post-process parameters the sleeping look consists of, and how
long sleep lasts — is supplied by the caller.

## State

```text
RECORD SleepEffector EXTENDS ScreenEffector
  target        : ParameterSet   # the fully-asleep look, supplied by the caller
  total         : real           # the effector's nominal lifetime
  attack_frac   : real in (0,1)  # fraction of the lifetime spent fading in
  release_frac  : real in (0,1)  # fraction at which fading out begins
  phase         : ENUM { beginning, faded_in, sleeping, waking }
```

**Invariants**

- The fade-in fraction is never zero and the fade-out fraction is never one: the first
  would make the blend weight divide by zero at the start, the second at the end. A
  caller supplying zero for either gets a half-and-half default instead, which is the
  safest thing the file can do without refusing.
- The phase is advanced partly by elapsed time and partly by the *game* — the sleeping and
  waking phases are entered from outside, not derived here. That split is why the phase is
  a field and not a computed value.

## `CSleepEffectorPP` (construction)

**Contract** — takes the fully-asleep parameter set, a nominal lifetime, and the fade-in
and fade-out fractions. A zero fraction is replaced by one half. Registers itself in the
sleep effector slot, so that a second sleep replaces the first rather than compounding
with it. Starts in the beginning phase.

## `Process`

**Contract** — called each frame by the camera's post-process pipeline. Computes how far
through its nominal life the effect is, derives a blend weight from that and from the
current phase, and writes the blend of the neutral look and the sleeping look into the
pipeline's parameter record. Always reports itself alive; it is removed by expiring or by
being replaced, never by its own decision.

```text
FUNCTION Process(out : ParameterSet) -> bool
  base.Process(out)                              # base lifetime countdown
  progress <- (total - remaining) / total        # 0 at start, 1 at expiry
  weight <- 1

  IF progress < attack_frac THEN
    weight <- progress / attack_frac             # fading in
    phase  <- beginning
  ELSE IF phase = beginning AND progress <= release_frac THEN
    weight <- 1
    phase  <- faded_in                           # fully asleep look reached; now wait
  ELSE IF phase = sleeping THEN
    remaining <- attack_frac * total             # FREEZE the clock: see note
    weight    <- 1
  ELSE IF phase = waking THEN
    weight <- (1 - progress) / (1 - release_frac)   # fading out

  weight <- clamp(weight, 0.01, 1)

  IF phase = sleeping THEN RETURN alive           # screen already black; write nothing
  out <- blend(neutral_look, target, weight)      # per parameter, componentwise
  RETURN alive
```

**Invariants**

- Freezing the clock works by *rewriting the remaining lifetime every frame* to the value
  it had at the end of the fade-in. The effector therefore sits permanently at that
  progress point, and when the phase leaves the sleeping state the countdown simply
  continues from there. This is the mechanism by which an effector with a fixed lifetime
  covers an arbitrary duration.
- While in the sleeping phase the routine returns without writing anything. The pipeline
  keeps the last written values, which are the fully-asleep look. Writing them again every
  frame would be identical work for an identical result; skipping is the point.
- The weight floor is a hundredth rather than zero, so the effect never fully releases the
  picture during a fade. A true zero at the start of the fade-in produces a one-frame snap
  as the pipeline switches between having and not having this effector.

**Notes** — the sleeping and waking phases are set by the actor's sleep logic, not here.
The order it must follow is: create the effector (the effect fades in), wait for the
beginning phase to become the faded-in phase, then set the sleeping phase to hold the
picture while the world clock advances, then set the waking phase to release it. Setting
the sleeping phase before the fade-in completes freezes a half-faded screen.

The blend is written out one parameter at a time — the two duality axes, the desaturation,
the blur, the three noise channels, and the three colour triples — each interpolated from
the neutral value to the target by the same weight. A rebuild with a parameter set that
knows how to interpolate itself writes one line here.

The noise frame rate is checked for zero on the way out for the same reason the generic
animator checks the grain: it is a divisor downstream and a zero has no diagnostic.
