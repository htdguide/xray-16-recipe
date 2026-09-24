# src/xrGame/HairsZone.cpp

> The "hairs" anomaly: a visible zone that wakes only when something inside it moves fast enough, then hits everything it holds in a random upward direction.

**Needs** — [`HairsZone.h`](HairsZone.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`ZoneVisual.h`](ZoneVisual.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md)
**Used by** — reached through its declarations in [`HairsZone.h`](HairsZone.h.md); callers name that, not this file.
**Tier floor** — T2: a proximity test and a hit per victim

## Purpose

An anomaly with a motion trigger. Standing still inside it is safe; moving is not. That
one rule is the whole design — it teaches the player that the correct response to this
anomaly is to slow down rather than to run — and it is expressed by overriding the base
zone's "should I wake up" test to consult each occupant's actual speed instead of its
mere presence.

The hit direction is deliberately random rather than radial, so repeated triggers do not
look like the same effect twice.

## State

```text
RECORD HairsZoneConfig
  min_speed_to_react : real   # a creature moving faster than this wakes the zone
```

## `Load`

**Contract** — reads the trigger speed from the section after the base zone loads.
Required, not optional.

## `CheckForAwaking`

**Contract** — the base zone's hook asking whether an idle zone should wake. Scans the
tracked occupants and wakes on the first *living* one whose measured movement speed
exceeds the threshold. Inert objects never wake it, so a corpse thrown in stays there
undisturbed.

```text
FUNCTION check_for_awaking()
  FOR EACH tracked object
    IF object is a living creature THEN
      IF object.movement.actual_speed > min_speed_to_react THEN
        switch_state(awaking)
        RETURN
```

**Notes** — the speed read is the creature's *actual* measured speed, not its intended
speed, so being pushed or falling triggers the zone as surely as running does.

## `Affect`

**Contract** — hits one victim with a randomly-directed upward-biased impulse and damage
falling off with distance from the zone's collision-sphere centre. Hits below a small
threshold are skipped along with their particle effect. An occupant flagged as ignored by
the zone is skipped entirely.

**Invariants** — the distance used for falloff is measured *horizontally*: the centre's
height is replaced by the victim's own before measuring, so an object directly above or
below the centre is treated as being at the centre. The anomaly is a column, not a
sphere.

```text
FUNCTION affect(object_info)
  target = object_info.object as a physics-shell holder
  IF target is none OR object_info.zone_ignore THEN RETURN

  centre = transform(collision_sphere.centre)   # the zone's own shape, not its origin
  centre.height = target.position.height        # flatten: the field is a column

  direction = normalize(random in [-0.5, 0.5], random in [0, 1], random in [-0.5, 0.5])
  # the vertical component is never negative: the anomaly always throws upward

  damage = power(distance(target.position, centre), radius)
  IF damage <= 0.01 THEN RETURN                 # rim of the zone does nothing at all

  issue_hit(victim: target, source: this zone, direction, damage,
            impulse: hit_impulse_scale * damage * target.mass,
            hit_type: blowout)
  play_hit_particles(target)
```

## `BlowoutState`

**Contract** — extends the base blowout tick by running the blowout's own visual update
while the base still reports the blowout as in progress. Unlike the gravitational
anomaly, this one does not run the per-object effect pass here; the base drives it.
