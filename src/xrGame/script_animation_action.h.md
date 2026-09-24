# src/xrGame/script_animation_action.h

> Declares the animation part of a script-issued action: play this clip, or just adopt this mental posture.

**Needs** — [`script_abstract_action.h`](script_abstract_action.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`script_animation_action_inline.h`](script_animation_action_inline.h.md)
**Used by** — [`script_animation_action_inline.h`](script_animation_action_inline.h.md) · [`script_animation_action_script.cpp`](script_animation_action_script.cpp.md) · [`script_entity_action.h`](script_entity_action.h.md)
**Tier floor** — T3: a small tagged record

## Purpose

One of the parts a script action is assembled from. It carries either an animation to play or
a mental state to adopt, and which of the two it is decides whether the action has to *wait*
for anything.

**Mental state** is the creature's posture toward the world — relaxed, wary, panicked — and it
selects which family of idle and locomotion animations the creature draws from. Setting it is
instantaneous. Playing a named clip is not.

The whole substance is in
[`script_animation_action_inline.h`](script_animation_action_inline.h.md); the script export
is in [`script_animation_action_script.cpp`](script_animation_action_script.cpp.md).

## State

```text
RECORD ScriptAnimationAction extends ScriptAbstractAction
  goal_type              : {ANIMATION, MENTAL} = MENTAL
  animation              : text                     # the clip's name; used when ANIMATION
  mental_state           : {FREE, DANGER, PANIC} = DANGER
  monster_anim_action    : monster animation kind = NONE   # the monster path; see below
  animation_index        : int = 0                  # which variant of that kind
  use_movement_controller: bool = false             # the clip drives the creature's position
  completed              : bool                     # inherited; see the invariants
```

**Invariants**

- `goal_type` is the discriminant and the record is a tagged union in all but name: setting
  an animation sets it to ANIMATION, setting a mental state sets it to MENTAL, and the
  unselected fields keep whatever they held. A rebuild should make it a real sum type.
- **A mental-state part is born completed; an animation part is born incomplete.** That
  single difference is what makes a script action wait for a clip and not wait for a posture,
  and it is decided in the setters rather than anywhere the action is run.
- The movement-controller flag is **reset to false by both setters**, so it must be set
  after whichever setter is used, not before. Only the two-argument constructor sets it
  together with an animation. This is a real trap for a rebuild that reorders construction.

## The monster path

A separate constructor takes a *kind* of monster animation — stand idle, sit idle, lie idle,
eat, sleep, rest, attack, look around, turn, capture-prepare — plus an index selecting among
that kind's variants. Monsters do not have named clips the way humans do; their animation
banks are addressed by role, so a script asks for "an attack animation" rather than for a
clip by name.

**Invariants** — that constructor leaves the goal type and the mental state at their
zero values rather than at their declared defaults, which means the goal type reads as
ANIMATION and the mental state as the first of its enumeration rather than as DANGER. Whether
the zeroing is deliberate is not recoverable; it happens to give the right goal type.

## `use_animation_movement_controller`

**Contract** — when set, the creature's position and orientation are driven *by the
animation* for the clip's duration, rather than the animation being played on top of
engine-driven movement. This is how scripted set pieces — climbing a ladder, a scripted fall,
an animation that must land the creature exactly on a mark — stay exactly aligned with the
world. A rebuild needs this switch: without it, a scripted sequence's root motion and the
engine's movement fight.

## Exported units

Four constructors (empty; animation name; animation name plus the movement-controller flag;
mental state; and the monster kind plus index), the two setters, and an `initialize` that
does nothing. See the inline twin.
