# src/xrGame/stalker_animation_manager_inline.h

> The animation manager's channel slots, script-queue operations, hook accessors, and the one predicate that decides whether the skeleton's blend tracks need advancing at all.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md)
**Used by** — [`stalker_animation_manager.cpp`](stalker_animation_manager.cpp.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md)
**Tier floor** — T3: field access and one disjunction

## Purpose

Mostly accessors, separated only because the original language wants inline bodies after the
class. Two of them are not accessors and are why this page is longer than eight lines.

## Channel accessors

**Contract** — `global`, `head`, `torso`, `legs` and `script` each hand back the
corresponding channel slot for direct manipulation. `object` returns the acting stalker,
hard-failing if it is absent; `data_storage` returns the shared animation table, likewise.

## `pop_script_animation`, `clear_script_animations`, `script_animations`

**Contract** — the scripted-animation queue. `pop` removes the front entry and resets the
script channel; `clear` empties the queue and resets the channel; the accessor exposes the
queue for the code that asks whether anything is pending. `pop` hard-fails on an empty
queue.

**Invariants** — both mutators **reset the channel as well as touching the queue**, and the
pairing is not optional. The channel holds a handle to the blend playing the front entry; a
queue edit that left the channel holding it would have the next frame believe a finished
animation is still running, and the queue would never advance.

## `non_script_need_update`

**Contract** — does any non-script channel have work this frame? True if the global channel
has been taken over by a hook, or if any of the four engine-driven channels reports that its
blend needs advancing.

```text
FUNCTION non_script_need_update() -> bool
  RETURN global_selector is set
      OR global_callback is set
      OR global.need_update() OR head.need_update()
      OR torso.need_update()  OR legs.need_update()
```

**Notes** — this is a cost gate, not a correctness one. Advancing the skeleton's blend tracks
is the expensive part of the frame for a stalker, and a stalker standing perfectly still
with no pending animation end needs none of it. The two hook checks are pessimistic on
purpose: a subsystem that has taken over the global channel may change its mind on any
frame, so its presence alone forces the update.

## Hook accessors

**Contract** — `global_selector`, `global_callback` and `global_modifier` read and write the
three hooks by which another subsystem takes over the global channel: one chooses the motion,
one is told when it ends, one gets to adjust the blend after it starts. Each is a plain
slot; setting one is how a subsystem claims the channel and clearing it is how it releases
it.

**Notes** — this is the extension point that lets smart covers and scripted sequences drive
a stalker's whole body without the animation manager knowing they exist. The three hooks are
separate because the subsystems that use them need different subsets: a sequence that only
needs to know when a motion ended should not have to supply a selector.

## `target_speed`, `special_danger_move`

**Contract** — `target_speed` reports the last non-zero speed the leg animation asked for,
which the movement system reads back to drive the body at the animation's own pace.
`special_danger_move` is a flag another subsystem sets to request the alternate danger
locomotion set.
