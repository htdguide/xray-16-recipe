# src/xrGame/stalker_animation_manager.h

> Declares the stalker's animation system: five independent channels — global, head, torso, legs, script — each holding one blend, resolved every frame in a fixed priority order.

**Needs** — [`stalker_animation_pair.h`](stalker_animation_pair.h.md) · [`stalker_animation_script.h`](stalker_animation_script.h.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`stalker_animation_manager_inline.h`](stalker_animation_manager_inline.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`xrAICore/Navigation/graph_engine_space.h`](../xrAICore/Navigation/graph_engine_space.h.md)
**Used by** — [`ai_stalker.cpp`](ai/stalker/ai_stalker.cpp.md) · [`ai_stalker_script_entity.cpp`](ai/stalker/ai_stalker_script_entity.cpp.md) · [`object_actions.cpp`](object_actions.cpp.md) · [`object_handler.cpp`](object_handler.cpp.md) · [`sight_manager.cpp`](sight_manager.cpp.md) · [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`stalker_animation_callbacks.cpp`](stalker_animation_callbacks.cpp.md) · [`stalker_animation_global.cpp`](stalker_animation_global.cpp.md) · [`stalker_animation_head.cpp`](stalker_animation_head.cpp.md) · [`stalker_animation_legs.cpp`](stalker_animation_legs.cpp.md) · [`stalker_animation_manager.cpp`](stalker_animation_manager.cpp.md) · [`stalker_animation_manager_debug.cpp`](stalker_animation_manager_debug.cpp.md) · _and 13 more_
**Tier floor** — T2: a declaration over five blend slots and per-frame selection state

## Purpose

Declares a surface implemented across six files, which is unusual enough to be worth naming
here: construction and reinitialization in
[`stalker_animation_manager.cpp`](stalker_animation_manager.cpp.md); the per-frame
resolution in [`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md);
one file per channel in [`stalker_animation_global.cpp`](stalker_animation_global.cpp.md),
[`stalker_animation_head.cpp`](stalker_animation_head.cpp.md),
`stalker_animation_torso.cpp` and
[`stalker_animation_legs.cpp`](stalker_animation_legs.cpp.md); and the aiming bone
callbacks in [`stalker_animation_callbacks.cpp`](stalker_animation_callbacks.cpp.md).

The original's own note says this type is five managers wearing one trenchcoat and should be
split. A rebuild should take that advice: the five channels share only the skeleton and the
frame, and every piece of shared state below is shared by exactly one pair of them.

## The five channels

```text
global  — a whole-body motion that suppresses every other channel while it plays
          (critical wounds, scripted whole-body actions, panic)
head    — the head's own motion; talking, listening, idle
torso   — the upper body: what the stalker is doing with its weapon
legs    — locomotion: direction, gait, turning in place
script  — a queued animation a script asked for; suppresses everything
```

Exactly one of `script`, `global` or the head/torso/legs trio plays in any frame, chosen in
that order. The trio plays together, partitioned by bone group, which is what lets a stalker
aim while walking.

## Exported units

- `update`, `reinit`, `reload` — the frame and lifecycle entry points.
- `global`, `head`, `torso`, `legs`, `script` — the five channel slots.
- `add_script_animation`, `pop_script_animation`, `clear_script_animations`, `script_animations` — the scripted-animation queue.
- `assign_global_animation` — the global channel's selection, public because the script surface can override it.
- `global_selector`, `global_callback`, `global_modifier` — hooks that let another subsystem take over the global channel entirely.
- `play_fx` — fire a one-shot additive reaction motion at a given strength.
- `play_delayed_callbacks` — run the animation-end callbacks deferred from the previous frame.
- `assign_bone_callbacks`, `assign_bone_blend_callbacks`, `remove_bone_callbacks`, `forward_blend_callbacks`, `backward_blend_callbacks` — the aiming bone rotations.
- `standing`, `target_speed`, `special_danger_move` — queries the movement system reads back.
- `object`, `data_storage` — the acting stalker and its shared animation table.
- `add_animation_stats` — checked-build instrumentation.

## Notes

The three aiming bones each carry a small parameter record naming the rotation to apply, the
stalker, an optional blend to fade against, and a direction. Those records are fields of the
manager rather than of the bones because the renderer's bone callbacks take an opaque
pointer and the manager is what guarantees its lifetime — the problem being solved is that
the pose is computed inside the renderer, after the game layer has returned.

The head-as-a-separate-bone-group path is behind a compile-time flag; with it off the head
channel is simply reset whenever the global or script channel plays. With it on, the head
keeps animating underneath them. See
[`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md).
