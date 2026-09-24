# src/xrGame/ai/monsters/group_states/group_state_home_point_attack_inline.h

> When the enemy stands somewhere a creature cannot walk, stop trying: fall back to a reserved
> cover spot on the enemy's side of the territory and watch the open ground.

**Needs** — [`group_state_home_point_attack.h`](group_state_home_point_attack.h.md) · [`../states/state_move_to_point.h`](../states/state_move_to_point.h.md) · [`../states/state_look_point.h`](../states/state_look_point.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../state.h`](../state.h.md) · [Level graph — chapter 14](../../../../xrAICore/README.md)
**Used by** — [`group_state_home_point_attack.h`](group_state_home_point_attack.h.md)
**Tier floor** — T3: a reachability test, a squad cover reservation, and a two-rung cycle

## Purpose

**Dead code**: written, compiled, never instantiated. It is documented in full anyway, because it
is the most complete statement in the chapter of a problem the shipped code solves less well — the
player standing on a rock.

A creature whose enemy is unreachable has no good move. Charging produces the creature pressing
into geometry; ignoring the enemy produces a creature that visibly does not care about being shot.
This state's answer is to withdraw to a *reserved* cover spot inside the territory, positioned on
the line from the enemy toward home, and alternate between sitting there and turning to face the
most exposed direction — which reads as a creature waiting the intruder out.

Compare the shipped answer in
[`../controller/controller_state_attack_inline.h`](../controller/controller_state_attack_inline.h.md):
stand still and turn toward the enemy. That is cheaper and less interesting.

## State

```text
RECORD GroupHomePointAttackState
  target_vertex                 : int    # the reserved cover spot
  skip_camp                     : bool   # no directional spot existed; do not bother looking around
  first_tick_enemy_inaccessible : int    # when the enemy first became unreachable
  last_tick_enemy_inaccessible  : int    # the most recent tick it was unreachable
  state_started                 : int
```

The two inaccessibility clocks implement a debounce in *both* directions, described below.

## `enemy_inaccessible`

**Contract** — a four-part test, any of which makes the enemy unreachable. Pure; no side effects.

```text
FUNCTION enemy_inaccessible() -> bool
  # the enemy is standing more than a metre off its own navigation vertex
  IF distance(enemy.position, navigation.position_of(enemy.vertex)) > 1   RETURN true
  # the enemy has left our territory
  IF NOT home.at_home(enemy.position)                                     RETURN true
  # the enemy's position is off the mesh entirely
  IF NOT navigation.valid_position(enemy.position)                        RETURN true
  # the enemy's recorded vertex is not a real vertex
  IF NOT navigation.valid_vertex(enemy.vertex)                            RETURN true
  RETURN false
```

**Notes** — the first clause is the one that catches the player on a rock, and it is the cleverest
line in the file. Every entity carries a navigation vertex — the mesh cell it is *associated*
with — and that association survives the entity leaving the walkable surface. So a player standing
on a crate still has a vertex, and it is the vertex of the floor beside the crate. Measuring the
gap between the entity's real position and its vertex's position is therefore a direct test of
"has this thing left the walkable world", at the cost of one distance computation. A rebuild that
tests only vertex validity will not detect the case at all.

The one-metre threshold is authored in code. It must be larger than the mesh's own vertical
tolerance and smaller than a step onto anything a player can climb; nothing records how it was
picked.

## `check_start_conditions`

**Contract** — accept immediately if the creature is outside its territory. Otherwise accept only
after the enemy has been continuously unreachable for three seconds, and forget the observation
once it has been reachable again for three seconds.

```text
FUNCTION may_start() -> bool
  IF NOT home.at_home()                RETURN true      # always go home first

  IF enemy_inaccessible()
    first_tick = first_tick OR now()
    last_tick  = now()
    RETURN now() - first_tick > 3000                    # three seconds of it, continuously
  ELSE
    IF last_tick AND now() - last_tick > 3000
      clear both clocks                                 # three seconds of reachability forgets it
  RETURN false
```

**Notes** — the two-sided debounce is what makes this usable. A player who hops onto a crate for
half a second does not trigger the withdrawal; a player who stays there does. And once the player
comes down, the creature does not immediately forget — it waits three seconds before resetting,
so a player hopping on and off does not repeatedly re-arm the timer from zero. Both halves are
needed and a rebuild that implements only the first gets a creature that withdraws from a jump.

## `check_completion`

**Contract** — finished if the looking rung was suppressed and we have reached the looking rung
anyway; not finished if we are still outside the territory and have not started camping; finished
if the enemy became reachable and we have been here at least five seconds; finished on arrival at
the reserved spot with movement stopped.

**Notes** — the five-second floor after the enemy becomes reachable again is a third debounce, in
the exit direction. Without it the creature would bounce straight back out of the withdrawal the
moment the player stepped down, and the whole sequence would read as a twitch.

## `reselect_state`

**Contract** — alternate: after running to cover, look at the open ground; otherwise run to cover.
A two-rung cycle with no exit of its own — the composite above it is what stops it.

## `setup_substates`

**The cover rung.** Take the direction from the enemy toward the home point, ask the home region
for a spot in its outer ring on that side, and **reserve it with the squad** so no other pack
member picks the same one. If no directional spot exists, fall back to any spot in the inner ring
and set the flag that skips the looking rung. Then run there, aggressively accelerated with
braking, stopping within 1 unit, never rebuilding the path, with the attack sound.

**Notes** — the squad reservation is the pack content of this state and it is released on both exit
paths. Without it, a pack withdrawing from an unreachable enemy stacks on one cover spot.

The *skip* flag on the fallback is a small, correct piece of judgement: the directional spot faces
the enemy, so looking around from it is meaningful; the inner-ring fallback does not, so the
creature just goes there and the composite ends.

**The looking rung.** Ask the cover system for the least-covered direction from where the creature
stands, look at a point 10 units along it, stand idle for 2 seconds with the attack sound at the
idle delay.

**Notes** — looking at the *most exposed* direction rather than at the enemy is what makes this
read as watchfulness rather than as staring. It is the same idiom the resting behaviour uses (see
[`group_state_rest_idle_inline.h`](group_state_rest_idle_inline.h.md)), with a different duration
and a different sound.

## `finalize` / `critical_finalize`

**Contract** — both release the squad's reservation on the chosen cover spot and clear the
inaccessibility clocks. Identical bodies; a reservation must be released on either exit.
