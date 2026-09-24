# src/xrGame/script_sound.cpp

> Creating a script-owned emitter, the three ways to start it, and what happens when a script drops one that is still playing.

**Needs** — [`script_sound.h`](script_sound.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`GameObject.h`](GameObject.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_sound.h`](script_sound.h.md)
**Tier floor** — T2: guarded calls onto the audio seam

## Purpose

The operations of the script sound that are not pure forwarding: construction, destruction,
the three play entry points, and the one reader that has to cope with a sound that was
never started. The tuning surface is in
[`script_sound_inline.h`](script_sound_inline.h.md).

## Construction

**Contract** — takes a sound name and an AI sound kind. Records whether audio is disabled,
resolves the name against the sound root of the virtual filesystem, and creates an emitter
only if a file is actually there. A missing file is reported as a **script error** and
leaves the object emitter-less but usable.

```text
FUNCTION construct(name, ai_sound_kind)
  silent    = NOT audio_enabled
  file_name = name
  IF a file exists at <sound root>/name with the sound extension THEN
    emitter = create emitter(name, effect channel, ai_sound_kind)
  ELSE
    log script error "file not found"        # and leave emitter empty
```

**Invariants**

- A missing sound must not stop the game. Modders reference sounds they have not shipped
  routinely, and every later operation on this object tolerates the empty emitter through
  the silent-or-present assertion.
- The existence check and the emitter creation both take the *bare* name; the extension and
  the root are the engine's, not the script's. A script therefore cannot reach outside the
  sound tree by naming a path, which is the check's second purpose.
- The emitter is created on the **effect** channel, never music or ambient. A script sound
  is a world event, and the player's effect volume applies to it.

## Destruction

**Contract** — if the emitter still has a live feedback channel, report a script error
naming the sound, then destroy the emitter regardless.

**Notes**

This is the type's one piece of enforced discipline and it is a *diagnostic*, not a fix:
the sound is cut mid-playback either way. It exists because the failure it catches is
otherwise silent and baffling — a script's local sound handle goes out of scope, a looped
ambient bed stops for no visible reason, and nothing in the log says why. A rebuild should
keep the report.

## `play` and `play_at`

**Contract** — both require an emitter or the silent flag, failing loudly otherwise with
the sound's name in the message, since by this point the script has ignored the
construction-time error. `play` starts the sound attached to the given entity, so it
follows that entity; `play_at` starts it at a fixed world position while still naming the
entity as its owner.

```text
FUNCTION play(object, delay, flags)
  REQUIRE emitter present OR silent  ELSE FAIL WITH "there is no sound: <name>"
  emitter.play(owner = object's client object, or none, flags, delay)

FUNCTION play_at(object, position, delay, flags)
  REQUIRE emitter present OR silent  ELSE FAIL WITH "there is no sound: <name>"
  emitter.play_at(owner = object's client object, or none, position, flags, delay)
```

**Invariants**

- The owner may be nothing. A sound with no owner is a world sound: it is still heard and
  still perceived, but no entity is blamed for it, which matters because a creature ignores
  sounds it made itself.
- The `delay` is in the audio seam's units and is honoured by the mixer, not by a timer in
  the game layer. A rebuild that implements delay with a scheduled callback will drift
  against the sound clock.

Unlike everything else in this type, these two fail hard rather than logging. The
justification is that a missing sound was already reported once at construction; a script
that plays it anyway is asking for a sound that provably does not exist.

## `play_without_feedback`

**Contract** — starts the sound with no feedback channel retained, taking the volume and
position by value rather than from the emitter's stored parameters. The object cannot
afterwards stop, move or retune the sound, and `is_playing` will not see it.

**Notes**

This is the cheap path, and its point is that the same handle can be fired repeatedly
without each play cancelling the last: the emitter is a *template* here rather than a live
voice. A script that wants twenty overlapping impacts uses one of these rather than twenty
handles.

## `position`

**Contract** — returns the emitter's current world position, or the zero vector with a
script error when the sound was never started.

**Notes**

The only reader that must cope with a sound that has an emitter but no live voice, because
position is a property of the *playing* instance and not of the emitter. The zero-vector
fallback follows the facade's convention: a position that failed answers the origin.
