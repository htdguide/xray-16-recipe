# src/xrGame/step_manager.h

> Declares the footstep scheduler and the two points a creature class overrides.

**Needs** — [`step_manager.cpp`](step_manager.cpp.md) · [`step_manager_defs.h`](step_manager_defs.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`Actor.h`](Actor.h.md) · [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`step_manager.cpp`](step_manager.cpp.md)
**Tier floor** — T2: a schedule, a per-clip state and two hooks.

## Purpose

Declares the surface implemented in [`step_manager.cpp`](step_manager.cpp.md).

Two structural facts are decisions rather than declarations. First, this is a **mixin**: a
creature class inherits it alongside its entity base, and the manager recovers the creature
by converting itself — which is why it has a construction step separate from its constructor
and why that step asserts the conversion succeeded. A rebuild composes instead: the manager
holds a reference to the creature it belongs to, supplied at construction, and the two-phase
dance disappears.

Second, the manager is driven entirely from outside. It is told when an animation starts and
told to update each frame; it never polls the animation layer. That is what lets it stamp the
start time exactly when the clip starts rather than a frame later, which the anti-drift
arithmetic in the update depends on.

## Exported units

- `reload(section)` — load the authored per-animation footfall schedule and find the foot
  bones. Once per creature.
- `on_animation_start(motion, blend)` — rebind to a new clip: stamp the start time, look up
  its schedule, reset the per-leg firing record, hand the clip to the limb controller.
- `update(is first person)` — one frame: fire due footfalls, advance the cycle, restart a
  looping clip without drift.
- `on_step()` — the hook a creature class overrides for a camera shake. Empty by default.
- `is_on_ground()` — the hook a creature class overrides to suppress steps while airborne.
  True by default.
- `get_foot_position(leg)` — the world position of one foot; used to place a dust particle.
- privately, the sound-selection state, which never repeats the previous sound on the same
  surface.
