# src/xrGame/ai/monsters/group_states/group_state_hear_danger_sound.h

> Declares the pack response to a frightening sound, implemented in
> [`group_state_hear_danger_sound_inline.h`](group_state_hear_danger_sound_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_hear_danger_sound_inline.h`](group_state_hear_danger_sound_inline.h.md)
**Used by** — [`dog_state_manager.cpp`](../dog/dog_state_manager.cpp.md) · [`group_state_hear_danger_sound_inline.h`](group_state_hear_danger_sound_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the composite that decides whether a creature that heard something alarming leads the pack
away from the sound or follows whoever is leading. Split from its body only because C++ splits
templates that way.

## `CStateGroupHearDangerousSound`

- **construct** — register three substates: take cover, follow the leader, and go home
- **enter** — nothing beyond the base contract
- **reselect_state** — the leader/follower decision
- **setup_substates** — fill in the parameters of the two movement substates
- **remove_links** — forward the destruction notice

Two constants are authored in the declaration itself: the radius within which a follower gathers
around its leader (20 world units) and how many times the navigation mesh is sampled looking for
a spot in that radius (5). Its private state is the chosen destination vertex. Contracts are in
the implementation twin.
