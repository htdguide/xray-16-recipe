# src/xrGame/ai/monsters/burer/burer_state_attack_tele_inline.h

> Sweep the space around the enemy, the creature and the midpoint between them for anything loose in a mass window, lift it, and throw it at the enemy's head — with a separate, unconditional sweep that snatches any live grenade nearby.

**Needs** — [`burer_state_attack_tele.h`](burer_state_attack_tele.h.md) · [`burer.h`](burer.h.md) · [`telekinesis.h`](../telekinesis.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`burer_state_attack_tele.h`](burer_state_attack_tele.h.md)
**Tier floor** — T2: proximity queries, mass and gravity filtering against the physics world, per tick

## Purpose

The burer's second signature attack, and the one that makes the level's clutter into ammunition. The behaviour splits cleanly in two: choosing and throwing ordinary debris, which is timed and paced; and intercepting grenades, which is opportunistic and runs on its own sweep regardless of what phase the attack is in.

Holding an object is the telekinesis ability's job, not this state's: the state hands objects to the ability with a raise speed, a hold height and a duration, and the ability runs each object's own little state machine. This state only decides *which* objects and *when* to let them go.

## State

See [`burer_state_attack_tele.h`](burer_state_attack_tele.h.md). Invariants:

- The candidate list is scratch: it is rebuilt by every sweep and consumed by every selection, and objects already held are excluded from it.
- The health recorded at entry is the abort trigger: *any* damage taken during the attack ends it.
- Grenade interception subscribes to each caught grenade's destruction so the creature's hold list is cleaned when it detonates.

Read from the creature: `tele_find_radius`, `tele_object_min_mass`, `tele_object_max_mass`, `tele_max_handled_objects`, `tele_min_distance`, `tele_max_distance`, `tele_time_to_hold`, `tele_max_time`, `tele_raise_speed`, `tele_object_height`, `tele_fly_velocity`. See [`burer.cpp`](burer.cpp.md).

Fixed in code: grenades are held at 2.5 units, raised at 3 units per second; every hold is given a 10-second lease; the state gives up after 6 seconds without a throw; an object must have been held steady for 1.5 seconds before it may be thrown; grenade sweeps run at most once a second.

## `initialize`

**Contract** — Resets the phase machine, selects and lifts the first batch of objects from the candidates the start check already found, records the entry health, sets the overall deadline, and blocks the script layer from taking the creature.

## `execute` — the phase machine

**Contract** — One tick. Sweeps for grenades first, unconditionally, then advances the phase machine, then faces the enemy.

```text
FUNCTION execute()
  handle_grenades()

  IF phase == started
     force clip: telekinesis, variant 0
     IF the lift has not been timed yet
        clip_end_tick = now() + length_of(telekinesis clip 0)
        lift_started  = now()
     ELSE IF now() > clip_end_tick
        phase = holding

  ELSE IF phase == holding
     force clip: telekinesis, variant 1
     execute_hold()

  ELSE IF phase == fire
     force clip: throw, variant 0
     throw the selected object at the enemy's head
     clip_end_tick = now() + length_of(throw clip)
     phase = wait_throw_end

  ELSE IF phase == wait_throw_end
     force clip: throw, variant 0
     IF now() > clip_end_tick
        phase = (any objects still held) ? holding : completed

  face_enemy()
```

## `execute_hold` — choosing the next throw

**Contract** — Decides, each tick of the holding phase, whether to throw and what.

```text
FUNCTION execute_hold()
  IF lift_started + tele_time_to_hold > now()        RETURN   # let the lift settle
  IF NOT can_see_enemy_right_now()                   RETURN   # do not throw blind

  FOR EACH held object
     IF it is in the "keep" state AND has been kept for over 1500 ms
        selected = it ; phase = fire ; RETURN

  IF nothing is held OR lift_started + 6000 ms < now()
     phase = completed
```

**Notes** — Two independent settling times gate a throw: the authored `tele_time_to_hold` for the batch as a whole, and a fixed 1.5 seconds per object. The per-object one exists because an object that has only just reached hold height is still swinging, and a throw then goes wide. The 6-second give-up is what stops a burer that lifted a batch of objects it can never throw — because the enemy stayed out of sight — from standing there indefinitely.

## `check_start_conditions`

**Contract** — Whether the attack may begin. Refuses if anything is already held, if the enemy is outside the authored distance window, or if a fresh sweep finds no candidates. Note that the check *performs the sweep* — it is not a pure predicate; the candidate list it fills is what `initialize` then lifts.

## `check_completion`

**Contract** — Five ways out, any one of which ends it: the enemy came inside the minimum distance, the enemy went beyond the maximum, the creature lost any health at all, the overall deadline passed, or the phase machine finished.

**Notes** — The health test has no threshold — a single point of damage ends the attack. That is what makes telekinesis the burer's *opening* move and the shield its answer to being shot: the two can never overlap.

## `FindObjects` — the sweep

**Contract** — Rebuilds the candidate list from three proximity queries and de-duplicates the result.

```text
FUNCTION FindObjects()
  clear the candidate list
  sweep_around(enemy position)
  sweep_around(own position)
  sweep_around(the midpoint between the two)
  sort and remove duplicates
```

**Notes** — Three centres, not one. Sweeping only around the creature would make it throw the clutter at its own feet; only around the enemy would fail in an empty room; the midpoint catches the corridor between them, which is where a fight usually has furniture. The sweep radius is the same authored value for all three, so the three spheres overlap heavily and the de-duplication is not optional.

## `FindFreeObjects` — the filter

**Contract** — One proximity query, filtered. An object qualifies only if it is a physics-simulated object with an active body, is not a creature, is not a grenade, is not tagged heavy in its own spawn configuration, has a mass inside the authored window, is not the burer itself, is not already held, and is actually subject to gravity.

**Notes** — Each exclusion is a distinct failure it prevents. Creatures are excluded because lifting one would fight its own movement. Grenades are excluded *here* because they are handled by their own path, which has different parameters and a destruction callback. The explicit "heavy" tag in an object's spawn configuration is a level-designer override for things that are within the mass window but should never fly — the mass window alone is not expressive enough. And the gravity test excludes objects already pinned or scripted into place, which would otherwise be lifted and snap back.

## `SelectObjects` — lifting a batch

**Contract** — Sorts the candidates by distance to the *enemy*, nearest first, lifts up to the authored maximum, and removes those from the candidate list.

```text
FUNCTION SelectObjects()
  count = min(candidates, tele_max_handled_objects)
  IF already holding more than `count`   RETURN
  sort candidates by distance to the enemy, ascending
  FOR the first `count` candidates
     height = tele_object_height
     IF the creature is configured as an indoor type
        height = height * 0.7
        rotate = false
     ELSE
        rotate = true
     hand the object to the telekinesis ability with
        (tele_raise_speed, height, 10000 ms lease, rotate)
     give it the hold and throw sounds
     mark it with the hold effect
  drop the lifted ones from the candidate list
```

**Notes** — Sorting by distance to the *enemy* rather than to the creature is the decision: the burer prefers ammunition that is already close to its target, because a shorter flight is harder to dodge. The indoor adjustment — lift to seventy per cent of the height and do not tumble the object — exists because a burer in a corridor would otherwise slam its own ammunition into the ceiling; the creature type is authored per configuration section, so a level designer chooses it.

## `HandleGrenades` — the interception

**Contract** — At most once a second, sweep for live grenades within the authored search radius of the creature and catch every one it can, up to one more than the ordinary object budget. Runs on every tick of the attack regardless of phase.

```text
FUNCTION HandleGrenades()
  IF now() < last_sweep + 1000 ms   RETURN
  FOR EACH nearby object that is a grenade with an active body,
           not already held, and subject to gravity
     subscribe to its destruction, to clear it from the hold list
     hand it to the telekinesis ability at (3 units/sec, 2.5 units, 10000 ms, no rotation)
     mark it with the hold effect
     IF held count >= tele_max_handled_objects + 1   BREAK
```

**Notes** — This is the burer's answer to being grenaded, and it deserves its own path for three reasons a rebuild must preserve. It is unconditional: it runs even while the creature is mid-throw, because a grenade cannot wait for a phase. Its hold height and raise speed are fixed rather than authored, because a grenade must clear the floor fast and does not need to be posed. And its budget is *one more* than the ordinary budget, so a burer already at capacity can still catch the grenade that would kill it.

The destruction subscription is what keeps the hold list clean when the grenade goes off in mid-air; without it the ability would keep a dead entry.

## `deactivate` — the teardown

**Contract** — Runs on both the normal and the aborted exit. Clears the candidate list, unsubscribes from every held grenade's destruction, stops the hold effect on every held object, *throws everything still held at the enemy*, releases the ability, and restores the script layer's reach.

**Invariants** — After this the creature holds nothing and has no outstanding subscriptions.

**Notes** — Ending the attack by flinging the remaining objects rather than dropping them is a design decision, not tidiness: it means a burer interrupted mid-telekinesis discharges its load instead of gently setting down a room's worth of furniture. `FireAllToEnemy` aims each object at the enemy's *head* position and derives each one's flight time from its own distance divided by the authored fly velocity, so a volley of objects at different ranges arrives together rather than in the order they were lifted — which is what makes the discharge read as one attack.

Only objects in the raising or holding state are thrown; ones already in flight are left alone.
