# src/xrGame/MosquitoBald.cpp

> An anomaly that does not throw anything: it simply damages everything inside it, hard at the blowout and continuously between them.

**Needs** — [`MosquitoBald.h`](MosquitoBald.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`Level.h`](Level.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)
**Used by** — [`MosquitoBald.h`](MosquitoBald.h.md)
**Tier floor** — T3: a per-object damage loop over the zone's contents

## Purpose

The base zone class models an anomaly as a cycle — idle, accumulate, blow out, discharge — and
leaves the actual effect to the subclass. This one's effect is the simplest possible: a hit,
scaled by distance from the zone centre, delivered in a random upward direction. What
distinguishes it is a second, weaker hit applied *between* blowouts, so standing in it is
steadily harmful rather than harmless until the next cycle.

## State

```text
RECORD MosquitoBald                    # on top of the base zone
  last_blowout_update : bool           # guards the once-per-transition blowout refresh
  # hit_impulse_scale is inherited and set to 1 here: the impulse equals power times mass
```

**Invariants** — `last_blowout_update` exists to make the blowout refresh run *once* on each
side of the state change rather than every tick, and it must be cleared whenever the zone is
not blowing out, or the next blowout is skipped.

## `BlowoutState`

**Contract** — the per-tick blowout hook. Runs the base behaviour, then refreshes the blowout
exactly once on entering the state and once on the tick it ends.

```text
FUNCTION BlowoutState() -> bool
  still_blowing = inherited BlowoutState()
  IF NOT still_blowing THEN
    last_blowout_update = false
    UpdateBlowout()                  # the final refresh as the state ends
  ELSE IF NOT last_blowout_update THEN
    last_blowout_update = true
    UpdateBlowout()                  # the first refresh as it begins
  END IF
  RETURN still_blowing
```

**Notes** — the refresh is what rebuilds the set of objects the blowout will affect. Doing it
twice — once at each edge — catches both the objects present when the blowout started and any
that entered before it ended.

## `Affect`

**Contract** — the blowout's effect on one object inside the zone. Damages objects with a
physics body, scaled by their distance from the zone's collision sphere centre, in a randomly
chosen upward-biased direction. Skips objects the zone has been told to ignore. Does nothing
when the computed power is negligible.

```text
FUNCTION Affect(entry)
  target = entry.object as a physics holder; IF none THEN RETURN
  IF entry.zone_ignore THEN RETURN

  centre = the zone's collision sphere centre, in world space
  # A random direction biased upward: horizontal components in [-0.5, 0.5], vertical in
  # [0, 1]. The bias is what makes hits look like the ground erupting rather than a shove.
  hit_dir = normalize(random(-0.5, 0.5), random(0, 1), random(-0.5, 0.5))

  # Distance measured to the object's SURFACE, not its origin, so a large object is hit
  # as hard as a small one standing in the same place.
  dist  = distance(target.position, centre) - target.radius
  power = falloff(max(dist, 0), zone radius)
  impulse = hit_impulse_scale * power * target.mass

  IF power > 0.01 THEN
    issue hit(target, source: this zone, hit_dir, power, bone: none,
              position: bone origin, impulse, type: the zone's blowout hit type)
    play the hit particles on the target
  END IF
```

**Invariants** — the impulse is proportional to the target's mass, which makes the resulting
*acceleration* independent of mass. Two objects of different weight are thrown the same
distance, which is what an anomaly should look like and is not what a physical impulse would
do.

## `UpdateSecondaryHit`

**Contract** — the continuous damage applied between blowouts. Walks every object inside the
zone and applies the same shape of hit at the configured secondary power, without particles.
Runs at most once per frame and never during precache.

```text
FUNCTION UpdateSecondaryHit()
  IF already run this frame THEN RETURN
  mark this frame
  IF the device is precaching THEN RETURN
  FOR EACH entry IN the objects inside the zone
    IF entry.object is being destroyed THEN CONTINUE
    target = entry.object as a physics holder; IF none THEN RETURN
    IF entry.zone_ignore THEN RETURN
    ... the same centre, direction and distance computation as Affect ...
    power = secondary_hit_power * relative_falloff(max(dist,0), zone radius)
    IF power < 0 THEN RETURN
    issue hit(..., impulse: hit_impulse_scale * power * mass, type: blowout hit type)
  END FOR
```

**Notes** — three of the guards inside the loop `RETURN` rather than `CONTINUE`, which
abandons the whole pass as soon as one object is uninteresting. Objects later in the list are
silently spared. This is a defect, not a decision, and a rebuild should continue.

The frame guard is the standard shape for a hook the zone machinery may call more than once in
a frame; the precache exclusion keeps the zone from damaging anything during the warm-up
frames before the player is really in the world.

The secondary hit deliberately plays no particles: it is a constant drain, and an effect on
every tick would be unreadable.
