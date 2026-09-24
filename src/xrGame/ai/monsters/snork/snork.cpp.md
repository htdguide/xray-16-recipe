# src/xrGame/ai/monsters/snork/snork.cpp

> A creature defined by three small things: it leaps, it senses the player through walls, and it snarls before a fight — but only half the time, and only once.

**Needs** — [`snork.h`](snork.h.md) · [`snork_state_manager.h`](snork_state_manager.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../control_movement_base.h`](../control_movement_base.h.md) · [`../../../Level.h`](../../../Level.h.md) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`snork.h`](snork.h.md)
**Tier floor** — T2: casts rays against the static collision database

## Purpose

Almost the whole creature is its configuration section and the animation table built from it.
Three things are code.

**The leap is a three-clip sequence, not a state.** Like the giant's stomp, the leap is a
motion-control ability the shared base offers; this file only supplies the wind-up, airborne
and landing clip names and the velocity profile to fly with. Nothing in the snork's brain
mentions jumping.

**The snarl is a coin flip that fires at most once per fight.** The brain sets a flag when it
first enters the attack state; the first threaten request afterwards consumes the flag and then
refuses half the time at random.

**Distant sense is one boolean.** Everything about how a snork finds the player through a wall
is shared code gated on the predicate in [`snork.h`](snork.h.md).

## `Load`

**Contract** — reads the animation table for every action the shared base can request, binding
each to a clip prefix, a velocity profile, a posture and a set of four directional reaction
effects. Registers three damaged-variant substitutions, so that while the "damaged" flag is set
the idle, walk and run requests silently resolve to their damaged clips and the brain never
learns the creature is hurt. Ends with the shared base's post-load pass. Fails hard on a
missing mandatory clip.

**Notes** — the two running turn animations are registered *optionally*, and under **two
alternative names each**. The registration attempts one prefix, then a second, and only if one
of them exists does it install the substitution that makes running turns use it. This is
compatibility machinery: the three games this engine loads ship the snork's model with
differently-named turn clips, and the creature silently does without the animation on data
where neither name is present. A rebuild that demands one name breaks on two of the three
shipped games.

Several action bindings collapse onto the idle clip — sleeping, resting and dragging all play
"stand idle". The snork has no authored animation for those, and rather than leaving the
request unbound (which would fail) they are aliased. A rebuild is free to make the aliasing
explicit in data.

## `reinit`

**Contract** — loads the landing velocity profile from the section, registers the leap as a
wind-up clip, an airborne clip and a landing clip with that profile, clears the one-shot snarl
flag, and binds the threaten animation with the fraction of the clip at which its effect lands.
Runs on every spawn and every save load.

**Notes** — the leap registration passes an explicitly absent identifier for one of its
optional slots and a zero for another, which is the shared jump machinery's way of saying "this
creature has no separate mid-air loop and no second velocity profile". The threaten hit
fraction (a little under two thirds of the clip) is the creature's contract with its own
animation.

The debug-only target vertex is cleared here under a comment reading "remove this" — a
developer convenience described below.

## `check_start_conditions` — the snarl gate

**Contract** — answers whether a motion-control request may seize the creature. Consults the
shared base first. For the threaten request only, applies the snork's own rule. Has a *side
effect*: it consumes the one-shot flag.

```text
FUNCTION check_start_conditions(kind) -> bool
  IF NOT base.check_start_conditions(kind)
    RETURN false
  IF kind == threaten
    IF NOT start_threaten          RETURN false
    start_threaten = false         # consumed whether or not the snarl happens
    IF random_percent() < 50       RETURN false
  RETURN true
```

**Invariants** — the flag is cleared *before* the coin flip, so a snork gets exactly one
chance to snarl per entry into combat and loses it half the time. That is the whole design: a
snork that snarled every fight would be predictable, and one that snarled repeatedly would be
tiresome. A rebuild that clears the flag only on success gets a snork that snarls every fight
after a variable delay, which is a different creature.

A predicate with a side effect is bad shape and worth fixing in a rebuild — but the *behaviour*
is deliberate and must survive the fix.

## `on_activate_control`

**Contract** — the request was granted; plays the threaten vocalisation through the creature's
sound player. A commented-out alternative would have played it in the player's head instead,
as the giant's does; the snork's stayed a world sound.

## `CheckSpecParams`

**Contract** — called by the animation layer when a clip carries a special-parameter marker.
Two markers are handled: a corpse-inspection marker queues the inspection clip as a one-shot
sequence, and a scared-idle marker switches the current animation to the look-around clip and
returns immediately. Bit-tested, so a clip may carry both — but the scared marker's early
return means the inspection still runs and the scared switch wins.

## `HitEntityInJump`

**Contract** — applies damage when a leap connects, reading power, impulse and impulse
direction from the *airborne clip's* own authored parameters rather than from the creature's
section. Damage that belongs to a move is authored with the move, which is how the same
creature's pounce and bite can differ without two sets of numbers in the section.

## `jump`

**Contract** — the script-facing leap: launches the creature at a world position with a
strength factor and plays the aggressive vocalisation. This is the only entry point by which
something outside the creature can make it jump, and it bypasses the ability gate entirely — a
script says jump and the snork jumps.

## `trace`, `trace_geometry`, `find_geometry` — the wall probe

**Contract** — `trace` casts one ray from the creature's centre against the *static* collision
database only, out to its radius plus thirty units, and reports the hit distance or "nothing
within range". `trace_geometry` casts four and answers whether they describe a flat surface.
`find_geometry` asks whether such a surface is within ten units ahead.

```text
FUNCTION trace_geometry(direction) -> bool, range
  range = trace(direction)
  IF range > 30                    RETURN false      # nothing close enough to be a wall

  # tilt up by the angle that subtends one unit at that distance, and re-probe
  angle = arcsin(1 / range)
  centre_dir = direction tilted up by angle
  range = trace(centre_dir);       IF range > 30  RETURN false
  Pc = centre + centre_dir * range

  # two more probes, offset half a body radius left and right of that point
  left_dir  = normalize((Pc + right_axis * radius/2) - centre)
  range = trace(left_dir);         IF range > 30  RETURN false
  Pl = centre + left_dir * range

  right_dir = normalize((Pc - right_axis * radius/2) - centre)
  range = trace(right_dir);        IF range > 30  RETURN false
  Pr = centre + right_dir * range

  # the three points lie on a plane if the two segment bearings agree
  RETURN bearing(Pl - Pc) ≈ bearing(Pc - Pr) within 0.1 radians on both angles
```

**Invariants** — the test is "three points on a line, seen from here", which is the cheapest
available proxy for "a flat vertical surface" and is what a creature needs in order to know it
could jump *onto* a wall rather than into a corner. Only static geometry is probed, so a
barricade of loose objects is invisible to it. Any probe missing entirely aborts the test —
absence of geometry is a negative answer, not a neutral one.

**Notes** — **this probe is dead.** Its only caller is a line inside the per-frame update that
has been commented out, and `find_geometry` is called from nowhere else in the engine. The
ten-unit and thirty-unit limits and the half-radius offset are compiled in, and the intent —
a wall-run or wall-pounce that was never finished — is not recoverable beyond what the geometry
of the test implies.

## `UpdateCL`

**Contract** — per-frame update. In a shipping build it defers entirely to the shared base. In
a developer build it additionally draws the nearest cover point to the player and a vertical
marker above the snork, which is a visualisation of the cover query the attack states use.

**Notes** — the developer build also binds a key that records the player's current navigation
vertex into the snork's debug target field. Nothing reads that field. It is a leftover from
whatever the wall probe above was being developed for.
