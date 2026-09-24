# src/xrGame/ai/monsters/controller/controller_state_attack_hide_lite_inline.h

> Run for cover, but stop the moment the enemy can no longer see you — the cheap variant of the
> controller's retreat.

**Needs** — [`controller_state_attack_hide_lite.h`](controller_state_attack_hide_lite.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`controller.h`](controller.h.md) · [`controller_animation.h`](controller_animation.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`controller_state_attack_hide_lite.h`](controller_state_attack_hide_lite.h.md)
**Tier floor** — T3: cover query plus path request

## Purpose

Identical in machinery to [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md)
and different in exactly two decisions, which is the whole reason it is a separate state:

1. **The goal is concealment, not the cell.** It finishes as soon as the enemy stops seeing the
   creature, even mid-route. The full version finishes only on arrival.
2. **It lets the pathfinder prefer covered cells along the way** instead of running the straight
   route to the chosen cell.

It also drops the sprint/demeanour handling entirely — this state never changes how the creature
carries itself.

**Currently unreachable in play**: no state manager registers it. It is a written-and-dead
alternative to the full retreat.

## State

```text
RECORD ControlHideLiteState
  target_position : vector
  target_vertex   : int
  time_finished   : int    # stamped on exit; never read
```

The finish stamp is written and never consulted anywhere. A rebuild can drop it.

## `CStateControlHideLite`

**Contract** — choose a cover vertex on entry and prime the path builder; each tick re-assert the
path target, the aggressive acceleration profile, the aggression sound, a head-look at the
enemy's remembered position, and the running body pairing. Finishes on arrival at the target
vertex with movement stopped, *or* as soon as the enemy no longer sees the creature. Always
willing to start.

```text
FUNCTION select_target()
  point = cover_system.find_cover(from = enemy_position, min_radius = 10, max_radius = 30)
  IF point EXISTS
    target_position, target_vertex = point.position, point.vertex
  ELSE
    target_vertex   = 0
    target_position = navigation.position_of(target_vertex)

FUNCTION execute()
  path.target        = (target_position, target_vertex)
  path.rebuild_every = 0
  path.stop_distance = 0
  path.use_covers    = true        # prefer concealed cells en route — the "lite" difference
  animation.accelerate(aggressive, braking = false)
  sound.play(aggressive, delay = section.attack_sound_delay)
  look_at(enemy_position)
  set_body_pairing(torso = run, legs = run)

FUNCTION is_finished() -> bool
  IF self.vertex == target_vertex AND NOT path.is_moving  RETURN true
  RETURN NOT enemy_sees_me()
```

**Notes** — the same vertex-zero fallback as the full retreat, with the same consequence; the
source keeps a commented-out assertion beside it, so the author knew the case was unhandled.

Note the asymmetry with the full retreat: it does not request a movement action at all. Whatever
action the previous state left in place carries over. That works only because the state is
unreachable; a rebuild that wires it up must add the run request.
