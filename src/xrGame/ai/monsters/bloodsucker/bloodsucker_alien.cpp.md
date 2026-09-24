# src/xrGame/ai/monsters/bloodsucker/bloodsucker_alien.cpp

> The camera takeover: the player's view is moved to the bloodsucker's head, his weapon is taken away, and the field of view widens as the creature runs.

**Needs** — [`bloodsucker_alien.h`](bloodsucker_alien.h.md) · [`bloodsucker.h`](bloodsucker.h.md) · [`ai_monster_utils.h`](../ai_monster_utils.h.md) · [`ActorEffector.h`](../../../ActorEffector.h.md) · [`controlled_actor.h`](../controlled_actor.h.md) · [`Inventory.h`](../../../Inventory.h.md)
**Used by** — [`bloodsucker_alien.h`](bloodsucker_alien.h.md)
**Tier floor** — T2: two unbounded per-frame effectors driving the camera basis and field of view

## Purpose

A scripted set piece rather than ordinary behaviour: for its duration the player is not
playing, he is watching. The takeover installs two effectors with unbounded lifetimes on the
player's camera chain, blocks his weapon, hides his crosshair, makes the creature invisible,
and — through the shared controlled-actor machinery — takes his movement.

It is worth a page because it is the chapter's only mechanism that *replaces* the player's
camera rather than perturbing it, and because the inertia model it uses is a genuinely
interesting piece of camera work.

## State

```text
RECORD AlienControl
  creature          : reference
  active            : bool
  camera_effector   : reference
  post_effector     : reference
  crosshair_was_on  : bool
```

**Invariants** — both activation and deactivation return immediately when already in that
state, so the pair is idempotent and safe to call from a state's setup every tick.

**Notes** — the two effectors are identified in the camera chain by an identifier derived
from the takeover object's own address. That is a cheap unique identifier, flagged as such in
the original; it is incidental and a rebuild allocates a real one. What it *records* is that
each takeover needs its own identity in the camera chain, because two bloodsuckers could in
principle do this at once.

## `activate`

**Contract** — installs the creature as the player's controller, ensures the player is the
creature's enemy, blocks the weapon, hides the crosshair (remembering whether it was on),
installs the two effectors, and makes the creature invisible outright. Requires a player.

```text
FUNCTION activate()
  IF already active THEN RETURN
  install self as the player's controller, with turning suppressed
  IF the creature has no enemy THEN make the player its enemy
  block the player's weapon entirely
  crosshair_was_on = the crosshair setting ; turn the crosshair off
  install the post-process effector (using the creature's vampire effector parameters)
  install the camera effector
  creature.invisible = true ; creature.stop_rendering()
  active = true
```

**Notes** — "turning suppressed" means the controlled-actor layer will not rotate the player
toward anything; the camera effector owns the orientation completely, so any other source
would fight it.

The post-process parameters are **borrowed from the vampire effector** rather than being
configured separately. A rebuild may want its own.

## `deactivate`

**Contract** — the exact inverse, plus restoring the crosshair only if it was on. Removes
both effectors from the camera chain by identifier and destroys the post-process one
explicitly.

**Notes** — the camera effector is removed by identifier and the reference dropped; the
post-process one is removed *and* then explicitly destroyed. The asymmetry reflects who owns
each in the camera chain, which is incidental — but the destruction routine sets its own
lifetime to zero and then deletes itself through a local alias, which is a real
self-destruction and a rebuild expresses it as the chain owning the object.

## the camera effector

**Contract** — replaces the player's camera basis entirely, each frame, with an inertially
smoothed copy of the creature's head transform, plus a slow random wobble, plus a
speed-dependent field of view. Never finishes on its own.

```text
FUNCTION process(camera)
  # 1. wobble: three independent angles, each easing toward a random target
  #    and picking a new one on arrival
  FOR EACH axis
    IF ease(current[axis], target[axis], 0.2 rad/s, dt) reached the target THEN
      target[axis] = a new random angle within ±10 degrees

  # 2. inertia: how hard the camera chases the head depends on how far it has moved
  target_transform = (position: creature's head, forward: creature's facing)
  relative_move = distance the head moved since last frame / 3.5, clamped to [0,1]
  ease(inertia, 1 - relative_move, at rate relative_move, dt)
  smoothed.position = smoothed.position eased toward target.position by inertia
  smoothed.forward  = smoothed.forward  eased toward target.forward  by inertia
  rebuild an orthonormal basis from the smoothed forward

  # 3. field of view widens with the creature's speed
  target_fov = 70 + (175 - 70) * clamp(creature speed / 15, 0, 1)
  ease(current_fov, target_fov, 80 degrees/s, dt)

  write the smoothed basis rotated by the wobble, and the field of view, into the camera
```

**Invariants** — the inertia coefficient is itself eased, and its target is *one minus* the
relative movement, at a rate equal to the relative movement. So a stationary head gives an
inertia approaching one (the camera locks on) and a fast-moving head gives an inertia
approaching zero (the camera lags), and the transition between them is itself faster when the
head is moving. That coupling is the whole feel of the effect: the view is glued to the
creature when it stalks and slews behind it when it lunges.

**Notes** — the field-of-view range, 70 to 175 degrees, is extreme by design: at a full
sprint the view is a fisheye. The speed normalisation divides by 15, which is well above any
bloodsucker's configured run speed, so the top of the range is never reached in practice —
the effective maximum is wherever the creature's run speed lands. Nothing derives 15, 3.5,
80, or the ten-degree wobble.

## the post-process effector

**Contract** — pulses a post-process state in and out forever, between 30% and 60% strength,
at a fixed rate. Never finishes on its own.

```text
FUNCTION process(accumulated)
  IF the factor has reached its target THEN
    target = (target > 0.5) ? 0.3 : 0.6     # ping-pong between the two bounds
  ease(factor, target, 0.3 per second, dt)
  accumulated = lerp(identity, state, factor)
```

**Invariants** — the effect never goes to zero or to full, which is what makes it read as a
slow breathing rather than as a flash. The ping-pong test is a comparison against the
midpoint of its own two bounds, so changing either bound without changing the test breaks it.
