# src/xrGame/ai/monsters/poltergeist/poltergeist_flame_thrower.cpp

> The flame poltergeist's attack: columns of fire raised at navigable points *near the player*, each running a three-phase lifecycle and burning anything the player-aimed ray passes through.

**Needs** — [`poltergeist.h`](poltergeist.h.md) · [`../ai_monster_effector.h`](../ai_monster_effector.h.md) · [`../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Networking transport](../../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: places emitters and casts a ray per flame per hit interval

## Purpose

This is the more interesting of the two poltergeist abilities because the attack does not come
from the creature at all. The creature stays hidden and drifting somewhere else; **flames erupt
around the player**, at points chosen on the navigation graph near the player's own vertex, and
each burns along a fixed direction aimed at the player's head *at the moment it was created*.

The consequence for play is that a flame is dodgeable — it does not track — and that several
may be alight at once, each on its own clock. The creature is a spawner, not a shooter.

Each flame runs the same three phases: **prepare** (a warning effect, no damage), **fire**
(the column, hitting on an interval), **stop** (a dissipating effect). Only the fire phase is
dangerous, and the prepare phase's duration is the player's window to move.

## State

```text
RECORD Flame
  target        : entity            # whose head this flame was aimed at
  position      : vector            # where the column stands; fixed for its life
  aim           : direction         # fixed at creation — the flame does not track
  phase         : enum { prepare, fire, stop }
  phase_started : int
  emitter       : optional<emitter> # the column; only during the fire phase
  sound         : sound
  last_hit_at   : int

RECORD PolterFlame                  # extends PolterAbility
  flames        : list<Flame>
  started_last  : int               # global clock at the most recent flame creation

  # authored — all required
  sound_name        : text   # "flame_sound"
  particles_prepare : text   # "flame_particles_prepare"
  particles_fire    : text   # "flame_particles_fire"
  particles_stop    : text   # "flame_particles_stop"
  prepare_duration  : int    # "flame_fire_time_delay"  — the dodge window
  fire_duration     : int    # "flame_fire_time_play"
  reach             : real   # "flame_length"
  hit_power         : real   # "flame_hit_value" at point blank
  hit_interval      : int    # "flame_hit_delay"
  max_flames        : int    # "flames_count" — how many may burn at once
  spawn_interval    : int    # "flames_delay" — minimum gap between creations
  place_min_dist    : real   # "flame_min_dist"   } the ring around the target
  place_max_dist    : real   # "flame_max_dist"   } a flame may stand in
  place_min_height  : real   # "flame_min_height" } the lift above the chosen vertex
  place_max_height  : real   # "flame_max_height" }
  aura_radius       : real   # "flame_aura_radius" — how near the player must be
```

Invariants: a flame's `aim` and `position` never change after creation. The emitter exists only
during the fire phase. The list never exceeds `max_flames`.

## `load`

**Contract** — reads the parameters above plus a whole second parameter set for a **scanner**
sub-ability: a radius, a delay range, a named screen-effect section whose seven fields and
three colour triples are parsed individually, and a scan sound.

**Notes** — the scanner is fully loaded and **never used**. No path reads its radius, its
delay, its effect or its sound, and its two state fields are initialised and never touched
again. It reads as an intended "the creature sweeps the area and the player's screen distorts
as the sweep passes" that was cut. A rebuild should not implement it; it should record that the
shipped configuration files still carry its keys, so a loader that rejects unknown keys will
fail on the original data.

The screen-effect section is parsed field by field with a text scan of three comma-separated
floats per colour. That the format is text rather than three keys is a frozen property of the
shipped configuration.

## `update_schedule`

**Contract** — the per-tick step: advance every flame's phase, apply damage from the ones that
are burning, reap the finished ones, then possibly create one more.

```text
FUNCTION update_schedule()
  base ability's scheduled work        # the idle vocalisation

  FOR EACH flame IN flames
    CASE flame.phase OF
      prepare:
        IF flame.phase_started + prepare_duration < now THEN enter_phase(flame, fire)

      fire:
        IF flame.phase_started + fire_duration < now
          enter_phase(flame, stop)
        ELSE IF flame.last_hit_at + hit_interval < now
          cast a ray from flame.position along flame.aim, length reach, against everything
          IF the nearest thing hit IS flame.target AND within reach
            damage = hit_power scaled down linearly to zero at full reach
            send a burn hit to the target, from the creature, along flame.aim,
                 with no impulse and no bone
            flame.last_hit_at = now

      stop:
        destroy this flame

  remove the destroyed flames from the list

  # --- create one more? ---
  IF the creature is alive
     AND flames held < max_flames
     AND the creature is not ignoring the player
     AND detection_level >= detection_threshold
    IF distance(player, creature) < aura_radius
       AND the creature's path builder considers the player's position reachable
      IF started_last + spawn_interval < now
        create_flame(targeting the player)
```

**Invariants** — a flame only damages the entity it was created for, and only when that entity
is the *nearest* thing the ray hits. Anything between them shields, which is what makes cover
work against the attack.

**Notes** — the target is always the player, hard-coded: the creature does not flame other
creatures even though the flame machinery is written against a general entity. That is a
deliberate scope reduction — the poltergeist is a set-piece encounter — and it shows up again
in [`poltergeist_telekinesis.cpp`](poltergeist_telekinesis.cpp.md).

Damage falls off linearly from full at the flame's base to zero at its reach, so standing at
the edge of a column is survivable.

The reachability test means a flame is never created for a player standing somewhere the
creature's own navigation cannot describe — which is how a player on a ledge escapes the
attack entirely.

The stop phase's dissipating effect is started when the phase is *entered*, and the flame is
destroyed on the very next tick that sees it in that phase, so the stop effect is fire-and-
forget and outlives the flame record. That is intended.

The hit is delivered through the engine's networked hit event rather than by calling the
target directly, which is how every damage source in this engine works and is not specific to
this ability.

## `enter_phase`

**Contract** — sets the phase, restamps its start, and performs the phase's one-off effect:
prepare plays the warning as a one-shot; fire starts the column emitter and keeps the handle;
stop destroys that handle and plays the dissipation as a one-shot.

## `create_flame`

**Contract** — finds a legal position, builds a flame there aimed at the target's head, starts
its sound, puts it into the prepare phase and restamps the spawn clock. Does nothing if no
legal position was found — and note that failing does **not** restamp the clock, so a failed
attempt is retried on the very next tick rather than waiting out the interval.

## `find_flame_position` — where a column may stand

**Contract** — picks a point near the target, on the navigation graph, reachable from the
target's own vertex in a straight line, lifted by a random height. Two stages.

```text
FUNCTION find_flame_position(target) -> optional<vector>
  FOR up to FIND_ATTEMPTS tries
    heading  = uniformly random around the full circle
    pitch    = the target's own pitch                  # see Notes
    offset   = a random distance within [place_min_dist, place_max_dist]
    candidate = target's vertex position + offset along (heading, pitch)

    vertex = the last navigable vertex walking from the target's vertex towards candidate
    IF such a vertex exists
      RETURN its position lifted by a random height in [place_min_height, place_max_height]

  # every random direction failed: fall back to the most OPEN direction, reversed
  angle  = the direction in which this vertex's high cover is LEAST, sampled every 30 degrees
  try once more along (angle + half a turn) with the same offset and walk
  RETURN that position lifted, or nothing
```

**Notes** — the directional walk, rather than a plain position test, is what keeps a flame from
appearing on the far side of a wall from the player: the walk stops at the last vertex reachable
in a straight line, so the column lands in the same room.

The pitch is taken from the target's facing and then a random heading is substituted, so the
column is placed on a cone tilted by wherever the player happens to be looking. Looking at the
floor tilts the whole sampling ring downward. That is near-certainly unintended — the pitch
should be level — and it is what the encounter was tuned against.

The fallback direction is the *least*-covered direction **plus half a turn**, which is to say
the most covered one. Placing the last-resort flame against the nearest wall is at least
defensible as "somewhere that definitely exists"; no rationale is recorded.

Five attempts, and a thirty-degree sampling step in the fallback, are unexplained.
