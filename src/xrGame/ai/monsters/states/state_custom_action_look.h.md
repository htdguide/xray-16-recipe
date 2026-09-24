# src/xrGame/ai/monsters/states/state_custom_action_look.h

> Declares the facing variant of the generic action leaf, implemented in
> [`state_custom_action_look_inline.h`](state_custom_action_look_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`state_data.h`](state_data.h.md) · [`state_custom_action_look_inline.h`](state_custom_action_look_inline.h.md)
**Used by** — [`monster_state_hear_int_sound_inline.h`](monster_state_hear_int_sound_inline.h.md) · [`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

The same slot as [`state_custom_action.h`](state_custom_action.h.md), with one addition: the leaf
also turns the creature to face a point while the action plays. That is what separates "look
around" from "look around *over there*", and it is used by the two sound-reaction behaviours.

Its parameter record extends the plain action record with a facing point. The extension is by
*addition* — the plain record's fields come first and the point after — which is what makes the
mismatch described in
[`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md) possible: a composite
that writes only the plain record leaves the point at whatever the leaf last held.

## `CStateMonsterCustomActionLook`

- **execute** — request the stored action and animation flag, turn toward the stored point, play
  the stored voice
- **is_finished** — after the stored timeout, or never if it is zero

Identical to the plain variant in every respect except the facing, and — unlike it — it routes the
action through the creature's own action-request operation rather than writing the animation
component directly. Contracts are in the implementation twin.
