# src/xrGame/ai/monsters/group_states/group_state_attack.h

> Declares the pack attack brain, implemented in
> [`group_state_attack_inline.h`](group_state_attack_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_attack_inline.h`](group_state_attack_inline.h.md)
**Used by** — [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`group_state_attack_inline.h`](group_state_attack_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the composite state that a pack creature uses when an enemy exists. Split from its body only
because C++ splits templates that way.

## `CStateGroupAttack`

- **enter** — arm the melee tracker, latch the enemy, claim or join a squad and publish an attack
  goal to it
- **execute** — the selector over eleven registered substates
- **setup_substates** — fill in the parameters of the four data-driven substates at the moment
  each is entered
- **leave** (clean and forced) — clear the enemy's aggression marks and drop any script-imposed
  enemy
- **remove_links** — forget the latched enemy when that object is destroyed
- **check_home_point** / **check_behinder** — the two latches the selector consults

Its private state is the latched enemy, three clocks (retreat cooldown, the two-stage
"enemy is behind me" tracker, and the threat-display clock), a per-activation random distance
offset, and a flag saying the threat display has run out of patience. Contracts are in the
implementation twin.
