# src/xrGame/ai/monsters/ai_monster_squad.h

> Declares the creature pack: the goals members report up, the commands the pack hands back down, and the shared claims that stop two creatures wanting the same thing.

**Needs** — [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md) · [`ai_monster_squad_attack.cpp`](ai_monster_squad_attack.cpp.md) · [`ai_monster_squad_rest.cpp`](ai_monster_squad_rest.cpp.md) · [`steering_behaviour.h`](../../steering_behaviour.h.md)
**Used by** — [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md) · [`ai_monster_squad_attack.cpp`](ai_monster_squad_attack.cpp.md) · [`ai_monster_squad_manager.cpp`](ai_monster_squad_manager.cpp.md) · [`ai_monster_squad_manager.h`](ai_monster_squad_manager.h.md) · [`ai_monster_squad_rest.cpp`](ai_monster_squad_rest.cpp.md) · [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_debug.cpp`](basemonster/base_monster_debug.cpp.md) · [`base_monster_think.cpp`](basemonster/base_monster_think.cpp.md) · [`burer_state_attack_inline.h`](burer/burer_state_attack_inline.h.md) · [`dog.cpp`](dog/dog.cpp.md) · [`dog_state_manager.cpp`](dog/dog_state_manager.cpp.md) · [`group_state_attack_inline.h`](group_states/group_state_attack_inline.h.md) · [`group_state_attack_run_inline.h`](group_states/group_state_attack_run_inline.h.md) · [`group_state_eat_eat_inline.h`](group_states/group_state_eat_eat_inline.h.md) · _and 16 more_
**Tier floor** — T3: two maps keyed by entity and a couple of claim lists

## Purpose

Declares the pack and its two vocabularies. The vocabularies are substance and are given
here rather than in the implementation, because they are the entire interface between an
individual creature's brain and the pack's coordination.

## The two vocabularies

```text
ENUM MemberGoal          # what a member reports it is trying to do
  attack_enemy           # carries: the enemy
  panic_from_enemy       # carries: the enemy
  interesting_sound      # carries: a position
  dangerous_sound        # carries: a position
  walk_graph             # carries: a navigation vertex — travelling somewhere
  rest                   # carries: a vertex and a position
  none

ENUM SquadCommand        # what the pack tells a member to do
  explore
  attack                 # carries: the enemy and an approach direction
  threaten
  cover
  follow                 # carries: the leader and a position to take up
  feel_danger
  explicit_action
  rest                   # carries: a position and a vertex
  none
```

**Invariants** — goals flow *up* (written by the member, read by the pack), commands flow
*down* (written by the pack, read by the member). A member is registered in both maps or
neither. Both records carry the same four payload slots — entity, position, vertex,
direction — and which are meaningful depends entirely on the type tag.

**Notes** — four of the nine command types (`explore`, `threaten`, `cover`,
`feel_danger`, `explicit_action`) are never issued by anything in this directory. They are
vocabulary the coordination was expected to grow into and did not. A rebuild may cut them,
at the cost of diverging from a name set that appears in debug output.

## Exported units

- **the pack** — the goal map, the command map, the leader, the cover and corpse claim
  lists, the danger-mode timer, and the coordination pass. See
  [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md).
- **the attack coordinator** — assigns approach directions around a shared enemy, and
  assigns each member an index within the pack. See
  [`ai_monster_squad_attack.cpp`](ai_monster_squad_attack.cpp.md).
- **the idle coordinator** — arranges followers around a travelling or resting leader. See
  [`ai_monster_squad_rest.cpp`](ai_monster_squad_rest.cpp.md).
- **pack grouping steering** — an adapter that presents the pack's members to the generic
  steering layer as "the neighbours to cohere with and separate from", so that a pack moves
  as a loose flock rather than as a column. It enumerates every member except the creature
  itself and reports its own position each tick. It is a pure adapter and decides nothing.
