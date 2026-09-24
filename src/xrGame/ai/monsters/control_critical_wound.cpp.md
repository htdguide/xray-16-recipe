# src/xrGame/ai/monsters/control_critical_wound.cpp

> The critical-wound collapse: the creature stops dead, plays one clip, and tells itself the state is over when it ends.

**Needs** — [`control_critical_wound.h`](control_critical_wound.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: seizes the body for the duration of a clip

## Purpose

A creature that takes a wound to a critical bone plays a collapse clip during which it is
helpless. Mechanically this is the minimal ability: seize, stop everything, play one clip,
end. It differs from every other ability in one respect, and that difference is the reason
it exists as its own element rather than as a sequencer use: **it tells the creature the
state has ended on release, not on the clip's end event.** So an ability that is aborted for
any reason — death, a script seizure, a level unload — still leaves the creature's wound
state correctly cleared.

## State

```text
RECORD CriticalWoundPayload
  animation : text    # a clip name, resolved on the creature's model
```

## `activate`

**Contract** — seize all four body resources, subscribe to animation end, stop the path, the
movement and the turn, resolve the named clip on the creature's model and start it.

## `on_release`

**Contract** — release the body, unsubscribe, and call the creature's
critical-wound-state-stop hook.

**Notes** — the hook on the release path rather than on the end event is the whole design.
A rebuild that moves it to the event handler will leave creatures permanently wounded
whenever the ability is interrupted.

## `on_event`

**Contract** — animation end raises the critical-wound-end event, which the creature's custom
manager turns into a release.

## `check_start_conditions`

**Contract** — the ability is not running and no other ability holds the body.

**Notes** — no test that the creature is actually wounded. The decision that a wound is
critical belongs to the creature's damage handling; by the time this ability is asked, it
has already been made.
