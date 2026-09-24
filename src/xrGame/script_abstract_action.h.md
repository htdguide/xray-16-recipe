# src/xrGame/script_abstract_action.h

> The one thing every script-issued action has: whether it is finished.

**Needs** — _(none)_
**Used by** — [`script_animation_action.h`](script_animation_action.h.md) · [`script_monster_action.h`](script_monster_action.h.md) · [`script_movement_action.h`](script_movement_action.h.md) · [`script_object_action.h`](script_object_action.h.md) · [`script_particle_action.h`](script_particle_action.h.md) · [`script_sound_action.h`](script_sound_action.h.md) · [`script_watch_action.h`](script_watch_action.h.md)
**Tier floor** — T3: an interface with one field

## Purpose

Scripts issue actions to creatures — move here, play this animation, say this line — and the
action list that carries them needs to know, uniformly, when each has run its course. This
declares that and nothing else, which is why it is a base for the whole family of script
action parts rather than a class anyone instantiates.

## State

```text
RECORD ScriptAbstractAction
  completed : bool = true     # starts TRUE
```

**Invariants** — **the default is completed.** An action part that nobody set up is already
done, so a script action carrying a part it did not configure does not block. Every part that
has real work to do clears the flag when it is initialized, not when it is constructed. A
rebuild that defaults to not-completed will hang every script action on its unused parts.

## `completed`

**Contract** — a plain read of the flag. Non-discardable: the whole point of asking is to act
on the answer.
