# src/xrGame/ai/monsters/states/monster_state_attack_camp_inline.h

> The ambush: a creature that can sense the player through walls takes cover well away from them, faces the most exposed direction, and cycles between watching and creeping toward where the player was last seen — until the player is actually visible, or close, or has hurt it.

**Needs** — [`monster_state_attack_camp.h`](monster_state_attack_camp.h.md) · [`monster_state_attack_camp_stealout.h`](monster_state_attack_camp_stealout.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_look_point.h`](state_look_point.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../../../cover_point.h`](../../../cover_point.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`monster_state_attack_camp.h`](monster_state_attack_camp.h.md)
**Tier floor** — T3: a cover query, a reservation, and a three-phase cycle

## Purpose

This is the behaviour that makes some creatures frightening to approach rather than to fight. It
has three distinctive parts: the admission test (which is also the cover choice), the
reservation that stops a pack from all camping the same spot, and the watch direction, which is
not toward the enemy.

## `check_start_conditions` — the admission test

**Contract** — whether the creature may begin an ambush. **Has a side effect**: on success it
records the cover point it found, which `initialize` then reserves.

```text
FUNCTION check_start_conditions() -> bool
  IF NOT creature.ability_distant_feel()   RETURN false
  IF no enemy                              RETURN false
  IF distance(me, enemy) < 20              RETURN false
  point = cover_manager.find_cover(from: enemy_position, min_radius: 10, max_radius: 30)
  IF point IS none                         RETURN false
  target_node = point.vertex
  RETURN true
```

**Invariants**

- **Only creatures that sense at range may ambush.** The predicate is the one that also grants
  perception of the player through walls, and the coupling is deliberate: a creature that has to
  *see* its target cannot usefully hide from it, because hiding would blind it. This is the
  single line that decides which creatures in the game ambush.
- **Twenty world units minimum.** An ambush from close range is just standing still, and the
  completion test below would end it immediately anyway.
- **The cover is searched around the *enemy*, in a ten-to-thirty-unit annulus.** The creature
  asks for somewhere the enemy cannot see, measured from the enemy's position — so the chosen
  point is near the enemy and concealed from it, not far away and safe. An ambush is set close.

## `initialize`, `finalize`, `critical_finalize` — the reservation

**Contract** — entering locks the chosen cover vertex against the creature's squad; both exits
unlock it. Both exits, not one: a creature displaced mid-ambush must release its claim or the
spot is lost to the pack for the rest of the level.

**Invariants** — the lock is what stops three creatures of a pack from converging on the same
cover point and standing inside one another. It is held for the whole ambush, including the
creeping phase, so the creature can return to it.

## `reselect_state` — the phase cycle

**Contract** — chooses the next phase from the one that just finished.

```text
FUNCTION reselect_state()
  CASE previous_child OF
    none        : select(approach_cover)      # first entry: go to the spot
    approach    : select(watch)
    watch       : IF creep_out.check_start_conditions()
                    select(creep_out)
                  ELSE
                    select(approach_cover)    # re-approach: re-plays the walk to the same node
    creep_out   : select(watch)
```

**Invariants** — the cycle is watch → creep → watch → creep, with a re-approach inserted whenever
creeping is not possible. Because the reserved cover vertex never changes, the re-approach is
usually a no-op walk that completes immediately and returns the creature to watching — which is
the intended idle loop: a creature that cannot creep out simply keeps watching.

## `setup_substates` — parameterising the two generic phases

**Contract** — hands the approach and the watch their parameters at selection.

The **approach** is told: go to the reserved vertex, running, with no timeout, stopping within
one unit ("get exactly to the point"), never rebuilding the route, accelerating aggressively
without braking, and playing the creature's authored idle vocalisation.

**Invariants** — *no rebuild* is the significant one. The route to the ambush spot is planned
once and not replanned, so a creature whose path is blocked mid-approach does not go around; it
fails and the phase ends. That is deliberate — replanning would make the approach wander
visibly, which defeats an ambush.

The **watch** is told: stand idle for ten seconds, facing a point ten units along the direction
the cover manager reports as *least covered*, with no turn delay and the idle vocalisation.

**Invariants** — **the creature does not watch its enemy. It watches the open ground.** The
direction comes from the cover system's per-vertex exposure data, which knows from which
bearing this position is most visible. So an ambushing creature faces the approach route rather
than the target, which is what makes it look like it is waiting rather than staring.

The ten-second watch is what paces the whole cycle.

## `check_completion` — the three ways an ambush ends

**Contract** — whether the ambush is over.

```text
FUNCTION check_completion() -> bool
  IF current_child == creep_out
    RETURN creep_out.check_completion()          # while creeping, the child decides

  IF current_child == watch
    IF I can see my enemy right now              RETURN true
    IF I have been hit since the watch began     RETURN true

  IF distance(enemy, me) < 5                     RETURN true   # checked in every phase

  RETURN false
```

**Invariants**

- **Being seen ends the ambush.** Visibility is the ambush's whole premise, so the moment it is
  available the creature stops hiding and attacks.
- **Being hit ends it.** The test compares the last hit time against the *watch phase's* start
  time, not the ambush's, so it only notices hits taken while actually watching — a hit taken
  during the approach does not abort it.
- **Proximity ends it in every phase**, including the approach, which is what stops a creature
  from walking past its enemy on the way to an ambush spot.
- The five-unit proximity test dereferences the enemy without checking one exists, which is safe
  only because the parent chain guarantees it.

## `check_force_state`

**Contract** — overridden to do nothing, deliberately suppressing the inherited preemption hook
so that nothing can clear the phase choice mid-ambush from underneath the cycle above.
