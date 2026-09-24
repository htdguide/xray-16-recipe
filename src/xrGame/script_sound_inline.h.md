# src/xrGame/script_sound_inline.h

> The pass-through half of the script-owned sound handle: every property a script can read or set on a playing sound, forwarded to the audio emitter.

**Needs** — [`script_sound.h`](script_sound.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`script_sound.h`](script_sound.h.md)
**Tier floor** — T2: thin forwarding onto the audio seam, but it is on the per-frame path for every scripted sound

## Purpose

A script sound is an emitter handle a Lua script owns by value: it constructs one from a
sound name, plays it attached to a game object or at a point, and then tunes it while it
plays. This file is the tuning surface — the accessors and the short-argument overloads.
The substantive operations (construction, the three play entry points, destruction) live
in [`script_sound.cpp`](script_sound.cpp.md).

The split exists only because the original wanted these bodies visible at every call
site; a rebuild should treat this file and its header as one type.

## State

`Stateless.` — every operation reads or writes the emitter held by
[`script_sound.h`](script_sound.h.md).

## Property accessors

**Contract** — reading frequency, minimum distance, maximum distance or volume returns
the emitter's current parameter; writing sets it on the live emitter, taking effect on the
next mix. Position, parameters-as-a-block and a tail sound to splice on at the end are
writable the same way.

**Invariants** — every one of these asserts that the object holds a real emitter *or* was
constructed in the silent mode the engine enters when audio is disabled by command line.
That is the whole reason the silent flag exists: the accessors must stay callable when
there is no device, because the shipped scripts call them unconditionally.

```text
FUNCTION set_min_distance(d)
  emitter.set_range(d, get_max_distance())    # the seam takes the pair, so the
                                              # other end is read back and resent

FUNCTION set_max_distance(d)
  emitter.set_range(get_min_distance(), d)
```

The two range setters are not independent: the audio seam accepts only a (min, max) pair,
so setting one re-sends the other. A rebuild whose audio layer has separate setters may
drop the read-back.

## `length`

**Contract** — the sound's total duration in **milliseconds**, floored from the seam's
seconds-valued answer. Scripts schedule follow-on events against this number, so the
truncation direction is load-bearing: a scheduled follow-up never lands after the sound
has already ended by one unit.

## `is_playing`

**Contract** — true while the emitter still has a live feedback channel in the mixer. This
is the one accessor that deliberately does *not* assert on a missing handle, because
scripts poll it after the sound has finished and with audio disabled it must answer false
rather than fail.

## `stop` and `stop_deferred`

**Contract** — immediate stop cuts the emitter at once; deferred stop lets the current
buffer finish and stops at the end of the sound, which is what callers use to end a looped
ambient bed without a click. Both are exported to script, and the deferred one under two
spellings (see [`script_sound_script.cpp`](script_sound_script.cpp.md)).
