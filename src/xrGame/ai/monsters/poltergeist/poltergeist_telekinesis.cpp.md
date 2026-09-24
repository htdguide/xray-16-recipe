# src/xrGame/ai/monsters/poltergeist/poltergeist_telekinesis.cpp

> The telekinetic poltergeist's attack: a four-phase cycle that lifts nearby physics objects one at a time, holds them, then hurls them one at a time at the player's head — but only objects that have a clear line to the player.

**Needs** — [`poltergeist.h`](poltergeist.h.md) · [`../telekinesis.h`](../telekinesis.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: hands a collision callback to the rigid-body layer that is invoked from inside its solver and must be released at a defined moment

## Purpose

The visible behaviour is the set piece: furniture, crates and debris rise silently into the air
around the player and then fly at him one after another. The mechanism is a four-phase cycle
on one clock, plus a selection rule that is almost all of the design.

The **selection rule** is what makes the attack readable rather than random. A candidate object
must be a physics object, not a creature, not marked heavy in its own spawn configuration,
within a mass band, affected by gravity, not already held — and, the interesting part, **a ray
from it to the player's centre or to the player's head must reach the player unobstructed**.
So the creature only lifts things it could actually hit the player with. A crate behind a wall
stays put.

Objects are lifted **one per interval**, not all at once, which is what produces the
poltergeist's slow accumulating threat; and they are fired **one per interval** too.

## State

```text
RECORD PolterTele                   # extends PolterAbility
  phase       : enum { start_raising, raising, firing, waiting }
  phase_at    : int                 # global clock at the phase's last advance
  next_gap    : int                 # milliseconds until the next per-object step

  # authored, all optional with defaults
  find_radius        : real   # "Tele_Find_Radius",                        10
  mass_min           : real   # "Tele_Object_Min_Mass",                    40
  mass_max           : real   # "Tele_Object_Max_Mass",                    500
  object_count       : int    # "Tele_Object_Count",                       10
  hold_duration      : int    # "Tele_Hold_Time",                          3000 ms
  wait_duration      : int    # "Tele_Wait_Time",                          3000 ms
  fire_gap           : int    # "Tele_Delay_Between_Objects_Time",         500 ms
  raise_gap          : int    # "Tele_Delay_Between_Objects_Raise_Time",   500 ms
  engage_distance    : real   # "Tele_Distance",                           50
  lift_height        : real   # "Tele_Object_Height",                      10
  hold_timeout       : int    # "Tele_Time_Object_Keep",                   10000 ms
  raise_speed        : real   # "Tele_Raise_Speed",                        3
  fly_speed          : real   # "Tele_Fly_Velocity",                       30
  collision_damage   : real   # "Tele_Collision_Damage",                   0.5
  sound_hold         : text   # "sound_tele_hold"   — required
  sound_throw        : text   # "sound_tele_throw"  — required
```

The held objects themselves live in the telekinesis mixin, not here; this file only drives it.

## `update_schedule` — the cycle

**Contract** — the per-tick step. Three gates before the cycle runs at all, then one phase
advance.

```text
FUNCTION update_schedule()
  base ability's scheduled work

  IF the creature is dead OR it is ignoring the player            THEN RETURN
  IF distance(player, creature) > engage_distance                 THEN RETURN
  IF detection_level < detection_threshold                        THEN RETURN

  CASE phase OF
    start_raising:
      IF phase_at + next_gap < now
        IF raise_one_object() failed THEN phase = raising    # nothing left worth lifting
        phase_at = now
        next_gap = raise_gap/2 + random below raise_gap/2
      IF still start_raising AND objects held >= object_count
        phase = raising ; phase_at = now

    raising:
      IF phase_at + hold_duration > now THEN BREAK           # hold them up there
      phase_at = now ; next_gap = 0 ; phase = firing
      FALL THROUGH

    firing:
      IF phase_at + next_gap < now
        fire_one_object()
        phase_at = now
        next_gap = fire_gap/2 + random below fire_gap/2
      IF nothing is held any more
        phase = waiting ; phase_at = now

    waiting:
      IF phase_at + wait_duration < now
        next_gap = 0 ; phase = start_raising
```

**Invariants** — the cycle only ever advances forward and always returns to waiting, so a
poltergeist that is interrupted mid-cycle resumes from wherever it was rather than restarting.
The raising phase ends either by reaching the object count or by running out of candidates —
both routes lead to the same hold.

**Notes** — every per-object interval is drawn as *half the authored value plus a random amount
up to half again*, so the authored number is the maximum and the mean is three quarters of it.
That jitter is what stops the objects rising and flying in a mechanical rhythm, and the
half-plus-half form is used identically for both the raise and the fire gaps.

The three gates are all-or-nothing: below the detection threshold the creature does not lift,
does not hold and does not fire, but it also does not *drop* what it already holds — the
objects stay suspended until the mixin's own hold timeout expires. A player who backs out of
range mid-attack leaves furniture hanging in the air.

The distance gate is measured against the *player*, so like the flamethrower this ability
targets only the player and no other entity.

## `find_candidates`

**Contract** — appends every liftable object within a radius of a position to a list. Called
three times per lift attempt, around three different centres. Does not deduplicate.

```text
FUNCTION find_candidates(out list, centre)
  FOR EACH object within find_radius of centre
    reject it unless ALL of:
      it is a physics object with an ACTIVE body
      it is not a creature
      its spawn configuration does not mark it heavy
      its mass is within [mass_min, mass_max]
      it is not the poltergeist itself
      it is not already held
      it is affected by gravity
    IF a ray from the object to the PLAYER'S CENTRE reaches the player
       OR a ray from the object to the PLAYER'S HEAD reaches the player
      append it
```

**Notes** — the "marked heavy" test reads a flag out of the object's own spawn configuration,
which is how a level author nails down a specific prop the creature must not throw — a mass
band alone cannot express "not this one".

Requiring gravity to affect the object excludes things already held by something else and
things pinned by the level.

Two rays are cast, to centre and to head, and either suffices. A crouching player behind a low
wall is still reachable at the centre; a standing one behind it is reachable at the head. The
pair is cheap insurance against a single unlucky ray.

## `raise_one_object`

**Contract** — gathers candidates from three centres, ranks them, and lifts **the single best
one**. Answers whether anything was lifted.

```text
FUNCTION raise_one_object() -> bool
  candidates = []
  find_candidates(candidates, player.position)
  find_candidates(candidates, creature.position)
  find_candidates(candidates, the midpoint between them)

  sort candidates by distance to the player, nearest first
  remove adjacent duplicates

  IF candidates is empty THEN RETURN false

  take the first
  hand it to the telekinesis mixin: raise at raise_speed to lift_height,
       hold for at most hold_timeout, without rotating
  give it the hold and throw sounds
  RETURN true
```

**Invariants** — the three search centres are the player, the creature, and the point halfway
between them, so the attack draws on debris along the whole line of engagement rather than only
from around the victim.

**Notes** — deduplication is done by removing *adjacent* equal entries after sorting by
distance, which only works because two entries for one object sort to the same place. It is
correct here and fragile; a rebuild should use a set.

The source keeps a disabled alternative that lifts every candidate up to the object count in
one call, and a disabled comparator that ranked by a three-way distance relation rather than by
plain nearness. The shipped behaviour is one object per call, ranked by nearness to the player.
The object count is then reached over several calls, spaced by the raise gap — which is exactly
what produces the rising-one-at-a-time effect. A rebuild that lifts them all at once has a
different creature.

Objects are lifted without rotation, so they hang level. That is a deliberate look.

## `fire_one_object`

**Contract** — finds the first held object that is still rising or being held, arms it with a
collision damage callback, and hurls it at the player's head, arriving at the authored flight
speed. Returns after the first one — this fires exactly one object per call.

```text
FUNCTION fire_one_object()
  FOR EACH held object
    IF its state is neither rising nor holding THEN CONTINUE
    aim   = the player's head position
    attach a one-shot collision callback carrying collision_damage
    hand it to the mixin: fly to aim over (distance to aim / fly_speed) seconds
    RETURN
```

**Notes** — the flight *time* is derived from a constant speed, so nearer objects arrive
sooner — the object travels at `fly_speed` regardless of range, which is what makes the attack
dodgeable at distance and near-unavoidable up close.

Aiming at the head is hard-coded to the player, and the source flags it as a limitation: the
machinery could aim at any enemy's head and does not.

The collision callback is the one genuinely low-level thing in this file. It is handed to the
rigid-body layer, invoked from inside that layer when the thrown object strikes something, sets
the damage fraction if the impact was above half the layer's own minimum, marks the impact as
already accounted for, and then **detaches itself** — so one thrown object damages once. The
self-detaching callback is the mechanism a rebuild must reproduce in whatever form its physics
layer offers; what must survive is: one throw, one damage event, at an authored fraction,
only for impacts above a threshold.
