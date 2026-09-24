# src/xrGame/ai/monsters/snork/snork.h

> Declares the snork: a leaping creature whose distinctiveness is the pounce, a coin-flip snarl before each fight, and the ability to sense the player through walls.

**Needs** — [`snork.cpp`](snork.cpp.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md)
**Used by** — [`snork.cpp`](snork.cpp.md) · [`snork_script.cpp`](snork_script.cpp.md) · [`snork_state_manager.cpp`](snork_state_manager.cpp.md)
**Tier floor** — T3: a creature type with two abilities and a few raycasts

## Purpose

Declares the surface implemented in [`snork.cpp`](snork.cpp.md). The snork is almost entirely
the shared base plus data: it adds no state to the brain, and its two declared abilities are
answered by one-line predicates the base consults.

## State

```text
RECORD Snork
  jump_velocities : two velocity profiles, prepare and landing   # authored
  start_threaten  : bool     # set by the brain when the attack state is first entered,
                             #   consumed by the next threaten request; see snork.cpp
  target_node     : int      # debug-only; written by a developer key binding, read by nothing
```

**Invariants** — `start_threaten` is a one-shot: the first threaten request after it is set
clears it whether or not the threaten actually happens. So a snork snarls at most once per
entry into combat.

## The abilities it declares

**`ability_jump_over_physics`** — true. Tells the shared base this creature may leap over
loose physics objects rather than routing around them, which is why a snork crosses a room of
scattered crates in a straight line.

**`ability_distant_feel`** — true. Tells the shared base this creature perceives the player at
range without line of sight. This is the snork's real distinguishing trait and it is one
boolean: everything downstream of it — the long-range approach, the camp-in-cover behaviour
that only creatures with distant sense can start — is shared code gated on this predicate.

**`run_home_point_when_enemy_inaccessible`** — false, overriding a base default. When a snork
cannot reach its enemy it does *not* retreat to its authored home point; it keeps trying. A
leaping creature is assumed to eventually find a way.

## Exported units

- `Load` — the animation table and its velocity profiles.
- `reinit` — the leap velocity profile, the leap's three-animation sequence, the threaten
  binding, and the one-shot snarl flag.
- `UpdateCL` — per-frame; in a shipping build it does nothing beyond the base.
- `CheckSpecParams` — reacts to animation-layer requests for a corpse inspection or a scared
  idle.
- `jump` — the script-facing leap.
- `HitEntityInJump` — damage when the leap connects.
- `check_start_conditions` — the snarl's coin flip.
- `on_activate_control` — plays the snarl.
- `find_geometry`, `trace`, `trace_geometry` — a wall-detection probe, described in
  [`snork.cpp`](snork.cpp.md) and called from nowhere live.
- `get_monster_class_name` — the name the script and configuration layers know it by.
