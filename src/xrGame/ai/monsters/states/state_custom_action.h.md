# src/xrGame/ai/monsters/states/state_custom_action.h

> Declares the generic parameterized action leaf, implemented in
> [`state_custom_action_inline.h`](state_custom_action_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`state_data.h`](state_data.h.md) · [`state_custom_action_inline.h`](state_custom_action_inline.h.md)
**Used by** — [`bloodsucker_predator_inline.h`](../bloodsucker/bloodsucker_predator_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](../bloodsucker/bloodsucker_predator_lite_inline.h.md) · [`bloodsucker_state_capture_jump_inline.h`](../bloodsucker/bloodsucker_state_capture_jump_inline.h.md) · [`burer_state_manager.cpp`](../burer/burer_state_manager.cpp.md) · [`group_state_custom_inline.h`](../group_states/group_state_custom_inline.h.md) · [`group_state_eat_inline.h`](../group_states/group_state_eat_inline.h.md) · [`group_state_rest_idle_inline.h`](../group_states/group_state_rest_idle_inline.h.md) · [`monster_state_controlled_follow_inline.h`](monster_state_controlled_follow_inline.h.md) · [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md) · [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md) · [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md) · [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md) · [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md) · [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md) · _and 5 more_
**Tier floor** — T3: a declaration

## Purpose

Names the most reused leaf in the whole monster behaviour tree. It is not a behaviour at all: it is
a *slot* that a composite fills with an action, an animation flag, a timeout and a voice, so that
"stand idle for two seconds making an idle noise" does not need its own class.

Nearly every composite in this directory registers at least one. The leaf owns a parameter record
and hands its address to the state base, which is how the composite's parameter-filling step writes
into it.

## `CStateMonsterCustomAction`

- **execute** — request the stored action, animation flag and voice
- **is_finished** — after the stored timeout, or never if that timeout is zero

The contract, and the ownership arrangement that makes the parameter block work, are in the
implementation twin.
