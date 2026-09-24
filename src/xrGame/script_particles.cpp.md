# src/xrGame/script_particles.cpp

> A script-owned particle effect: its transform, its optional path animation, and the two-way link that lets either side die first.

**Needs** — [`script_particles.h`](script_particles.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrEngine/ObjectAnimator.h`](../xrEngine/ObjectAnimator.h.md)
**Used by** — [`script_particles.h`](script_particles.h.md)
**Tier floor** — T2: transform composition and a lifetime protocol

## Purpose

Implements both halves declared in [`script_particles.h`](script_particles.h.md). Most of
the methods are one-line forwarding; three things here are decisions a rebuild must make
the same way: the ownership protocol between the handle and the world-side effect, the
derivation of emitter velocity from a path animation, and the fact that the handle keeps
its own transform rather than reading one back.

## State

```text
RECORD ScriptParticlesHandle           # what the script holds
  transform : matrix                   # the authoritative placement; identity at birth
  effect    : optional<WorldEffect>    # cleared when the world side dies first

RECORD WorldEffect EXTENDS ParticleEffect
  owner    : optional<ScriptParticlesHandle>   # cleared when the handle dies first
  animator : optional<PathAnimator>            # created lazily by load_path
```

**Invariants**

- The two references are a *mutual weak link*: exactly one side owns nothing, and whichever
  dies first clears the other's pointer to it. Neither may be followed without checking.
- The handle's `transform` is the source of truth for placement. Every aiming and moving
  method writes the transform and then pushes it to the world side; nothing ever reads a
  transform back out of the particle system. This is what makes `last_position` cheap and
  correct even while the effect is not playing.
- The world-side effect is created with the *not self-removing* flag, because the handle
  owns it. An effect that deleted itself on finishing would leave the handle holding a
  dangling reference — the mutual link exists precisely to make that impossible.

## The ownership protocol

**Contract** — three entry points, one per way the pair can come apart.

```text
FUNCTION handle.destroy                   # the script dropped its reference
  IF effect is none THEN RETURN           # the world side already went
  effect.release_owner                    # tell it not to call back
  effect.destroy                          # then take it down
  effect = none

FUNCTION world.internal_delete            # the particle system is reclaiming us
  IF owner is present THEN owner.effect = none
  <base behaviour>

FUNCTION world.destroy                    # explicit teardown, from either side
  IF owner is present THEN owner.effect = none
  <base behaviour>

FUNCTION world.release_owner
  REQUIRE owner is present                # only the owner calls this, and only once
  owner = none
```

**Invariants**

- The handle clears the back-reference **before** destroying the effect, not after.
  Destruction re-enters the world side's own teardown, which would otherwise write through
  a pointer to a handle that is mid-destruction.
- Both world-side endings clear the owner's pointer, because the particle system may
  reclaim an effect on its own schedule — a level unload, a budget cull — without anyone
  asking. A rebuild that handles only the explicit path will hand scripts a stale handle
  after a level transition.

**Notes**

This protocol is the reason the type is split in two at all. A single object cannot both
be owned by a script value whose lifetime the engine cannot see and be registered with a
world subsystem that may reclaim it. A rebuild with a weak-reference facility should use
one and delete both halves of this dance.

## Path animation

**Contract** — an effect can be attached to an authored motion path. `load_path` creates
the animator on first use and reloads it only when the requested path differs from the one
already loaded; `start_path`, `pause_path` and `stop_path` drive it and all three require
a path to have been loaded first.

```text
FUNCTION load_path(name)
  IF animator is none THEN animator = new path animator
  IF animator has no path OR animator.path_name IS NOT name THEN
    animator.clear
    animator.load(name)        # skipped when already on this path: reloading would
                               # restart a running animation mid-flight
```

## Velocity derived from the path

**Contract** — each scheduled update, an effect carrying a running path animation advances
the animation, then pushes both the new transform and the velocity implied by the move into
the particle system.

```text
FUNCTION scheduled_update(elapsed_milliseconds)
  <base behaviour>
  IF animator is none THEN RETURN
  dt        = elapsed_milliseconds / 1000
  previous  = animator.transform.translation
  animator.advance(dt)
  velocity  = (animator.transform.translation - previous) / dt
  push(animator.transform, velocity) into the particle system
```

**Invariants** — the velocity is differentiated from the animation, not read from it.
Emitted particles inherit it, which is what makes a moving effect trail correctly instead
of leaving a stack of stationary puffs. A rebuild that pushes the transform without the
velocity produces a visibly different effect.

The update is *scheduled*, not per-frame, so `dt` is whatever the scheduler granted and can
be several frames' worth. Differentiating over that interval is deliberate and is why the
division uses the granted interval rather than the frame time.

## Placement and aiming

**Contract** — five methods write the handle's transform and push it to the world side:

- `play` — start, wherever the transform currently puts it.
- `play_at(position)` — move the transform to the position, push it, start, then **push it
  again**. The second push is not redundant: starting the effect resets the particle
  system's notion of the emitter, and without it the first frame of particles appears at
  the origin.
- `move_to(position, velocity)` — translate the transform and push it with an explicit
  velocity, the manual equivalent of what the path animation does automatically.
- `set_direction(direction)` — build an orientation whose forward axis is the given
  direction, completing it to an orthonormal frame, keep the existing translation, push.
- `set_orientation(yaw, pitch, roll)` — the same, from Euler angles.

**Invariants** — both aiming methods **replace** the rotation and **preserve** the
translation, by building a fresh orientation and then writing the old translation into it.
They also push with zero velocity, so aiming a moving effect drops its inherited velocity
for one update. That is a wart, not a design; a rebuild should carry the last velocity
through.

## Start and stop

**Contract** — `stop` ends emission and kills particles already alive; `stop_deferred` ends
emission and lets live particles finish their lifetimes. Both leave the effect reusable:
`play` afterwards restarts it. `is_playing` and `is_looped` ask the particle system, since
looping is a property of the authored effect, not of this handle.
