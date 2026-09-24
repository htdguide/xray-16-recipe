# src/xrGame/script_sound_action.cpp

> Naming the sound a channel will play, and deciding on the spot that a missing one is already finished.

**Needs** — [`script_sound_action.h`](script_sound_action.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Virtual filesystem](../xrCore/README.md)
**Used by** — reached through its declarations in [`script_sound_action.h`](script_sound_action.h.md); callers name that, not this file.
**Tier floor** — T2

## Purpose

One method that needs the virtual filesystem, plus a destructor that releases nothing —
the channel owns no emitter, only a name.

## `set_sound`

**Contract** — records the sound's name, tags the goal *attached*, and checks that the
named sound actually exists in the sound tree. A missing sound is reported as a script
error and the channel is marked **both started and completed**.

```text
FUNCTION set_sound(name)
  sound_name = name
  goal_type  = attached
  IF a file exists at <sound root>/name with the sound extension THEN
    started   = false
    completed = false
  ELSE
    log script error "file not found"
    started   = true         # never hand it to the audio seam
    completed = true         # and report done immediately
```

**Invariants**

- The existence check happens **when the channel is built**, not when it plays. A missing
  sound is therefore reported once, at the moment a script constructs the action, where the
  message is attributable — rather than later from inside the scheduler where nothing
  identifies the culprit.
- Marking a missing sound *completed* is the load-bearing part: an action whose sound
  channel never finishes blocks its whole queue, so a typo in a sound name would freeze the
  entity. Declaring the impossible order already done lets the rest of the action run. A
  rebuild must reproduce this, or a missing asset becomes a hang instead of a log line.
- The started flag is set *as well*, so nothing later tries to play the sound that is not
  there.

**Notes**

The same check appears in [`script_sound.cpp`](script_sound.cpp.md) for the script-owned
sound handle, with the opposite resolution: there a missing file leaves a usable-but-silent
object, here it fabricates a completed order. Both are right for their context, and the
duplication of the lookup is incidental.

## Destruction

**Contract** — releases nothing. Stated because the sibling
[particle channel](script_particle_action.cpp.md) has the same empty destructor for a very
different and much less comfortable reason.
