# src/xrGame/stalker_danger_by_sound_actions.cpp

> An unfinished branch: five differently-named actions with one identical body, none of which the planner can reach.

**Needs** — [`stalker_danger_by_sound_actions.h`](stalker_danger_by_sound_actions.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`object_handler.h`](object_handler.h.md)
**Used by** — [`stalker_danger_by_sound_actions.h`](stalker_danger_by_sound_actions.h.md)
**Tier floor** — T2: one movement configuration per action entry.

## Purpose

The danger system classifies threats into four kinds, and this is the fourth: a sound with
no other corroboration. The intended behaviour is legible from the five action names —
stop and listen, go and check, take cover, look out, look around — and none of it was
written. All five actions share one body, and the classifier that would route a threat here
answers false unconditionally (see
[`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md)), so
the branch is unreachable in a shipped build.

It is written up rather than skipped because a rebuilder needs to know that the reaction is
*absent*, not that it is somewhere else. A stalker in the original does not react to an
unexplained sound as a distinct case; the sound of an enemy is folded into the
danger-in-direction branch, and everything else is ignored.

## State

`Stateless.`

## The shared body

**Contract** — all five actions have the same entry, and empty per-cycle and exit hooks.
The entry puts the creature in a wary standstill: stand where it legally can, do not move,
keep the eyes where they already point, and idle whatever is in hand.

```text
FUNCTION initialize()                  # identical in all five actions
  base.initialize()
  movement.desired_direction := none
  movement.path_type         := level path
  movement.detail_path_type  := smooth
  movement.head_for_nearest_accessible_position()
  movement.body_state        := standing
  movement.gait              := stand still
  movement.mental_state      := danger
  sight                      := keep the current direction
  weapon_goal(IDLE)
```

**Notes** — this is the neutral alert pose and nothing more. It is what an action looks like
before anyone has written its behaviour: legal position, no movement, no aiming, no sound.
A rebuild implementing this branch would keep the entry as the common base and give each of
the five a real per-cycle body; a rebuild omitting the branch loses nothing that is
observable in the original.

## Could not recover

Which of the five was meant to run first, and what would have distinguished "listen to"
from "check", cannot be recovered from the source. The planner wires only one of them and
calls it "fake".
