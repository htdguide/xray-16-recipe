# src/xrGame/AmebaZone.cpp

> An anomaly that slows everything living inside it and, when it discharges, throws them upward.

**Needs** — [`AmebaZone.h`](AmebaZone.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`ZoneVisual.h`](ZoneVisual.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`xrPhysics/PHUpdateObject.h`](../xrPhysics/PHUpdateObject.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`AmebaZone.h`](AmebaZone.h.md); callers name that, not this file.
**Tier floor** — T2: participates in the physics step, before the solve

## Purpose

Most anomalies are event-driven: they charge, they discharge, they hurt whatever is inside.
This one is different in that it also applies a **continuous constraint** — a speed limit on
every living creature inside it — and a constraint cannot be applied from the game's own
update, because the physics solver would immediately undo it. So this anomaly registers as
a participant in the physics step and applies the limit in the window between collision
detection and the constraint solve.

That registration is the load-bearing thing about the file. Everything else is a standard
anomaly.

## State

```text
  velocity_limit : real   # maximum speed a living creature may reach inside, read from
                          # the section's "max_velocity_in_zone"
```

## `Load`

**Contract** — reads the speed limit after the base anomaly has read its own tuning. The key
is required.

## `PhTune`

**Contract** — called by the physics world once per step, *before* the solve. For every
living creature currently inside, if it is within the effective radius, its movement
system's speed limit is set. The limit is set every step and is not restored when the
creature leaves.

**Invariants** — this must run inside the physics step. Applying a speed limit from the game
update would be overwritten by the solve, and applying it after the solve would produce a
visible stutter as the creature accelerated and was clamped each frame.

**Notes** — nothing ever *clears* the limit. A creature that walks out of the anomaly keeps
the reduced speed until something else sets it. Whether the movement system resets it per
step is not visible from this file, and a rebuild should verify it rather than assume.

## `BlowoutState`

**Contract** — the discharge. Asks the base anomaly whether the discharge has finished; if
it has not, advances the base discharge animation. Then, **unconditionally and every
update**, applies the discharge effect to every object inside.

**Invariants** — applying the effect on every update rather than once at the discharge
moment is what makes this anomaly a sustained throw rather than a single impulse. It is also
why an object can be lifted repeatedly during one discharge.

## `Affect`

**Contract** — the per-object effect: a hit whose direction is randomized in a **hemisphere
biased upward**, with a magnitude falling off with distance from the centre and an impulse
proportional to that magnitude and to the object's own mass.

```text
FUNCTION affect(object)
  IF the object has no physics body THEN RETURN
  IF the object is flagged to be ignored by anomalies THEN RETURN

  direction = normalize(random in [-0.5,0.5] x [0,1] x [-0.5,0.5])
              # the vertical component is never negative: it always throws UP
  power   = falloff(distance_to_centre, effective_radius)
  impulse = impulse_scale * power * object.mass    # mass-scaled: everything flies alike

  IF power > 0.01 THEN
    send a hit event: from this anomaly, at the object's origin, with the
      anomaly's own damage type
    play the hit particle effect on that object
```

**Invariants** — the impulse is scaled by mass so that a heavy object and a light one are
thrown at the same speed. That is a deliberate departure from physical realism and is what
makes the anomaly read as a force field rather than a push.

**Notes** — the small power threshold suppresses the effect for objects at the very edge,
which otherwise would receive a continuous stream of near-zero hits, each with its own
particle effect.

## `SwitchZoneState`

**Contract** — registers with the physics world on entering the discharge state and
unregisters on leaving it. The anomaly participates in the physics step only while
discharging.

**Invariants** — the registration is edge-triggered on both sides and must be, because
double registration or a missed unregistration leaves a destroyed anomaly in the physics
world's participant list.

## `distance_to_center`

**Contract** — the distance from an object to the anomaly's centre, where the centre is the
anomaly's collision sphere transformed into the world.

**Notes** — **the implementation is wrong** and reproducing it is a judgement call. It
expands the distance formula by hand and squares the *x* difference twice instead of using
*y* once: the vertical and depth axes are ignored entirely, so the measured distance is
`sqrt(2) * |dx|`. Every distance-based decision in this anomaly — the falloff, the speed
limit's radius test — is therefore effectively a slab test on one horizontal axis. A rebuild
that computes a true distance will make this anomaly behave noticeably differently from the
shipped one.
