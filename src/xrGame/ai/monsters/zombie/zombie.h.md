# src/xrGame/ai/monsters/zombie/zombie.h

> Declares the zombie — the creature whose defining trick is that it plays dead and gets back up.

**Needs** — [`basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`controlled_entity.h`](../controlled_entity.h.md) · [`ai_monster_bones.h`](../ai_monster_bones.h.md) · [`anim_triple.h`](../anim_triple.h.md) · [`zombie.cpp`](zombie.cpp.md)
**Used by** — [`zombie.cpp`](zombie.cpp.md) · [`zombie_script.cpp`](zombie_script.cpp.md) · [`zombie_state_manager.cpp`](zombie_state_manager.cpp.md) · [`script_game_object_use2.cpp`](../../../script_game_object_use2.cpp.md)
**Tier floor** — T3: an animation table, a health-band counter and four three-phase animations

## Purpose

Declares the surface implemented in [`zombie.cpp`](zombie.cpp.md).

The zombie is the one creature whose signature behaviour lives outside its behaviour tree.
Feigned death is driven from the **hit handler**, not from a state: taking fire below a
health threshold drops the zombie into a wind-up / hold / recovery animation that suppresses
its behaviour tree entirely for five seconds. The behaviour tree does not know about it and
does not need to.

## State

```text
RECORD Zombie
  bones                  : BoneChain          # head and spine look-at rig (dead; see zombie.cpp)
  head_bone, spine_bone  : BoneHandle

  fake_death_animations  : [4] AnimationTriple  # four authored variants of the fall
  active_variant         : optional<int>

  fake_death_count       : int    # how many feigned deaths this individual gets, rolled at load
  fake_death_left        : int    # how many remain
  health_death_threshold : real   # the health below which the first feigned death may trigger

  time_dead_start        : optional<int (milliseconds)>  # when the current feign began
  time_resurrect         : int (milliseconds)            # when the last feign ended
  last_hit_frame         : int                           # at most one feign trigger per frame
```

**Invariants** — `fake_death_count` is rolled *per individual at load time*, so two zombies
from the same configuration section do not get the same number of feigned deaths. That is
the whole reason the mechanic reads as unpredictable rather than as a rule.

## Exported units

- **construction, load, reload, reinitialise** — creature set-up; reload attaches the four
  feigned-death animation triples by name.
- **the hit handler** — where feigned death is decided.
- **the scheduled update** — where a feigned death ends.
- **fall down / stand up** — the same mechanic exposed for a caller to drive directly.
- **bone assignment and the bone callback** — a head-and-spine look-at rig that is built and
  never installed.
- **aiming and animation overrides** — the zombie refuses pitch correction on its aim and
  asks to be aimed at by its centre rather than by a bone.
- **the class name**, and the script registration hook.
