# src/xrGame/stalker_combat_action_base.cpp

> What every combat action shares: when a burst is allowed, how long it is, what the creature shouts, and how it is sent to cover.

**Needs** — [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`object_handler.h`](object_handler.h.md) · [`sound_player.h`](sound_player.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`inventory_item.h`](inventory_item.h.md)
**Used by** — [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md)
**Tier floor** — T2: reads the creature's tuned numbers and drives its weapon and sound managers.

## Purpose

Four different concerns collapse into this one base because every combat action needs all
four and none of them is worth a class: the aim-then-fire gate, the burst shape, the
squad-aware voice lines, and the "go stand there" that understands both kinds of cover.

## State

`Stateless.` Every helper reads the creature's tuned parameters and its current enemy
afresh; nothing is cached across cycles, because the enemy may be reselected between them.

## `initialize`

**Contract** — on entering any combat action, silence the creature's ambient chatter.
Concretely: every currently playing sound whose category is not the low-level humming
channel is stopped. Entering combat mid-sentence and finishing the sentence is the
behaviour this prevents.

## `finalize`

**Contract** — on leaving a combat action, clear the sound mask so that ordinary,
non-combat lines are allowed again — but only if the creature is still alive. A dead
creature keeps whatever mask it died with, because its death sound must not be unmasked
into the middle of a death animation.

## `fire`

**Contract** — the gate between aiming and shooting. If the creature's head is not yet
pointed close enough at the enemy, it does not fire; it re-asserts the aim goal and returns,
so the next cycle re-tests. Otherwise it issues a fire goal with a burst shape chosen for
the current range.

```text
FUNCTION fire()
  direction := enemy.position - self.position
  wanted_yaw := horizontal angle of direction
  IF angular_difference(wanted_yaw, head.current.yaw) > START_FIRE_ANGLE THEN
    aim_ready()          # keep turning; do not shoot down the wrong bearing
    RETURN
  params := select_queue_params(distance_to(enemy))
  weapon_goal(FIRE, best_weapon, params)
```

**Invariants** — the creature never issues a fire goal while its head is outside the fire
cone. The check is on the *head* orientation rather than the body's, because the weapon
follows the head.

**Notes** — the fire cone is an eighth of a half-turn, about 22.5 degrees, to either side.
It is wide: the point is not accuracy (the weapon's own dispersion decides that) but to stop
a stalker firing a burst at ninety degrees to its target during the turn. A tighter cone
makes creatures hesitate visibly while turning.

## `select_queue_params`

**Contract** — given the range to the target, produce four numbers: the minimum and maximum
number of rounds in a burst, and the minimum and maximum pause between bursts. Pure; reads
only the creature's tuned parameters and the class of its best weapon. The caller then
randomizes within each range, which is why ranges and not values are returned.

The shape is a two-level table lookup, and *both* levels are load-bearing:

```text
FUNCTION select_queue_params(distance) -> BurstShape
  class := weapon_class_of(best_weapon)     # pistol, shotgun, sniper, automatic, unknown
  # submachine gun and machine gun share the automatic-weapon row
  band := IF distance > class.far_distance    THEN far
          ELSE IF distance > class.med_distance THEN medium
          ELSE close
  RETURN class.band.(min_size, max_size, min_interval, max_interval)
```

**Invariants** — the two distance thresholds are per weapon class, not global; a pistol's
"far" begins much closer than a sniper rifle's. A rebuild that hoists the thresholds out of
the class loses the behaviour that makes pistol users close the distance and sniper users
hold it.

**Notes** — every number in the table comes from the creature's configuration section, not
from code. There are five weapon classes times three range bands times four numbers, and
naming them individually in the configuration is how the original lets each creature type —
a raw recruit, a veteran, a military squad — fight differently with the same code. A
rebuild should keep the table *in data*, indexed by (class, band), rather than reproduce
sixty individually named settings; the flat naming is an artifact of a configuration reader
without nested tables, not a decision.

Any weapon whose class is not one of the four named falls into the automatic row. That
default matters: unarmed and exotic weapons still get a plausible burst shape rather than
zeros.

## `aim_ready` and `aim_ready_force_full`

**Contract** — issue a weapon goal that brings the weapon to the aimed posture at the
current target, with the burst shape already chosen for the current range so that a fire
goal issued on a later cycle does not have to re-derive it. The `force_full` variant asks
for the full aiming animation rather than the abbreviated one; actions that begin from a
visible pose (stepping out of cover, an ambush opening) use it so the transition reads
correctly, while actions already holding the weapon up use the short form.

## `fire_make_sense`

**Contract** — asks the creature whether firing at the currently selected enemy would
accomplish anything: a live enemy, a usable weapon, a clear enough line. Pure delegation to
the creature; the combat actions call it before committing to a firing branch so that the
"cannot shoot" case is decided once.

## `play_attack_sound`

**Contract** — the shout on engaging. Suppressed entirely for non-human enemies (you do not
taunt a mutant), and suppressed when the squad coordinator says this member may not speak
right now — which is how a squad avoids six stalkers shouting simultaneously.

```text
FUNCTION play_attack_sound(window, id)
  IF enemy is not human THEN RETURN
  IF NOT squad.may_speak_noninformative() THEN RETURN
  IF squad.combat_member_count > 1 THEN
    line := IF squad.known_enemy_count > 1 THEN attack_with_allies_many_enemies
            ELSE attack_with_allies_one_enemy
  ELSE
    line := attack_alone
  sound.play(line, window, id)
```

**Notes** — the line varies by *squad composition and enemy count* rather than by the
speaker, which is the cheapest possible way to make combat chatter sound situational: the
same recorded line library produces "I've got him" when alone and "they're everywhere" when
outnumbered, with no dialogue authoring.

## `play_start_search_sound` and `play_enemy_lost_sound`

**Contract** — the two lines at the other end of a fight: beginning to search for a lost
enemy, and giving up on one. Both are gated on the squad speech permission and both pick
between an alone variant and a with-allies variant on the same combat-member count.
Neither is gated on the enemy being human, because both are addressed to allies rather than
to the enemy.

## `play_panic_sound`

**Contract** — the morale-break shout. Gated on nothing but the enemy's kind: the line
differs for a human enemy and a monster, and panic is never suppressed by the squad speech
permission because panic is exactly the moment the creature stops coordinating.

## `setup_cover`

**Contract** — send the creature to a cover point. Two quite different destinations hide
behind one call, and the cover point itself carries which:

```text
FUNCTION setup_cover(cover)
  IF cover is a smart cover THEN
    movement.target_params.cover_id := cover.id    # the movement manager will plan
    RETURN                                         # entry, loophole choice and animation
  movement.destination_vertex := cover.level_vertex
  movement.desired_position   := cover.position
```

**Invariants** — a smart cover is addressed by identity and an ordinary cover point by
position plus navigation vertex. The two are not interchangeable: an ordinary point is
somewhere to stand, whereas a smart cover is an authored place with entry animations,
loopholes and firing arcs, and reaching it means running the smart-cover sub-planner rather
than the pathfinder alone. A rebuild must keep the discriminator on the cover record; a
smart cover approached as a position lands the creature next to the cover instead of in it.

**Notes** — the ordinary branch sets both a navigation vertex and a free-floating position.
The vertex is what the pathfinder routes to; the position is the exact offset within that
vertex's cell where the creature should end up, which is what makes stalkers press against
the corner of a wall rather than stand in the middle of the nearest walkable square.
