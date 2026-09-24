# src/xrGame/script_movement_action_inline.h

> The movement channel's setters and its simpler constructors — and the rule that any change reopens the channel.

**Needs** — [`script_movement_action.h`](script_movement_action.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

Bodies for most of what [`script_movement_action.h`](script_movement_action.h.md)
declares. Two decisions live here rather than in the state description: which setters
change the goal tag, and the completion rule they all share.

## Every setter reopens the channel

**Contract** — each of the eleven setters writes its field and then clears the channel's
completion flag.

**Invariants**

- A channel that has been touched is by definition not finished. This is what lets a
  script retarget a running order — write a new position onto the channel an entity is
  already executing and the entity resumes rather than reporting done.
- The flag is cleared even when the value written is the one already there. A rebuild that
  optimizes the no-op write away changes behaviour: scripts re-set an unchanged field
  precisely to reopen a completed channel.

## Setters that also set the goal tag

**Contract** — three of the eleven decide what kind of goal the channel carries, in
addition to writing their field:

```text
set_patrol_path(path, name)  -> goal_type = patrol_path
set_position(position)       -> goal_type = path_position
set_input_keys(keys)         -> goal_type = input
```

The remaining setters — posture, movement type, path type, speed, the three patrol
modifiers — adjust *how* the goal is pursued and leave the tag alone. `set_object_to_go`
also sets the tag and lives in
[`script_movement_action.cpp`](script_movement_action.cpp.md) because it must unwrap a
script facade.

## Simple constructors

**Contract** — four constructors that are nothing but a run of setter calls, so the tag
they leave behind is whichever their last tag-setting call produced:

- (posture, movement type, path type, object, speed) — a goal of kind *object*.
- (posture, movement type, path type, position, speed) — a goal of kind *path position*.
- (held keys, speed) — a goal of kind *input*.
- monster (move action, position, stop distance, speed parameter) and monster
  (move action, object, stop distance, speed parameter) — the two monster forms that need
  no extra derivation.

## `initialize`

**Contract** — does nothing. The channel has no per-execution state to reset; it is
declared because the action aggregate resets every channel uniformly and needs the name to
exist on all of them.
