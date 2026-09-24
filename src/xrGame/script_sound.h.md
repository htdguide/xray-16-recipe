# src/xrGame/script_sound.h

> Declares the script-owned sound: an audio emitter a script constructs by name, plays, tunes and destroys by value.

**Needs** — [`script_sound.cpp`](script_sound.cpp.md) · [`script_sound_inline.h`](script_sound_inline.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`script_sound.cpp`](script_sound.cpp.md) · [`script_sound_action.h`](script_sound_action.h.md) · [`script_sound_action_inline.h`](script_sound_action_inline.h.md) · [`script_sound_inline.h`](script_sound_inline.h.md) · [`script_sound_script.cpp`](script_sound_script.cpp.md)
**Tier floor** — T2: a handle onto the audio seam

## Purpose

Declares the surface implemented in [`script_sound.cpp`](script_sound.cpp.md) and
[`script_sound_inline.h`](script_sound_inline.h.md). The type is the script layer's only
way to hold a sound across frames — the game object facade can *fire* a sound, but only
this type lets a script start a looped emitter, move it, retune it and stop it later.

## State

```text
RECORD ScriptSound
  emitter   : AudioEmitter       # the seam's handle; may be empty when audio is disabled
  file_name : text               # the authored sound's name, kept for error messages only
  silent    : bool               # audio was disabled at construction, so every operation
                                 # must succeed while doing nothing
```

**Invariants**

- Every operation asserts that the emitter exists **or** the silent flag is set. The flag
  is not an optimization; it is what keeps shipped scripts running with audio turned off at
  the command line, since they play sounds unconditionally.
- `file_name` is only read when reporting a failure. A rebuild that carries no diagnostic
  text may drop it.
- The handle is owned by value by a script variable. There is no engine-side registry, so
  the emitter dies with the script's last reference, and an emitter still playing at that
  moment is a script bug — reported, not prevented. See
  [`script_sound.cpp`](script_sound.cpp.md).

## Exported units

- construct from (sound name, AI sound kind).
- destroy.
- `play(object, delay, flags)` and its shorter forms — attached to an entity.
- `play_at(object, position, delay, flags)` and its shorter forms — at a world point, with
  the entity still named as the emitter's owner.
- `play_without_feedback(object, flags, delay, position, volume)` — fire and forget.
- `length`, `is_playing`, `stop`, `stop_deferred`, `attach_tail(name)`.
- Position, frequency, volume, minimum and maximum distance, and the whole parameter block:
  readers and writers. Documented in
  [`script_sound_inline.h`](script_sound_inline.h.md).

**Notes**

The **AI sound kind** given at construction is not an audio parameter. It is the
perception attribute the [feel](../../GLOSSARY.md) system reads when this sound reaches a
creature: what kind of event it was and therefore how a listener should react. A sound
constructed without one is audible but invisible to the senses system, which is the default
and the reason `no_sound` is spelled the way it is.

The sound action channel is declared a friend of this type so it can copy the sound's name
out of it — see [`script_sound_action.h`](script_sound_action.h.md). That is the only
reason the field is not simply public.
