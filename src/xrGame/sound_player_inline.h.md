# src/xrGame/sound_player_inline.h

> The sound player's queries, plus the two mask operations that are the only way whole categories of a creature's sounds are silenced.

**Needs** — [`sound_player.h`](sound_player.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`sound_player.h`](sound_player.h.md)
**Tier floor** — T2: linear scans over the live-sound list, called per frame

## Purpose

Separated from the declaration for C++ compilation reasons; fold into the type in a
rebuild. Two of the definitions here carry decisions rather than being accessors, and one
of them is genuinely surprising, so this file is not as thin as its size suggests.

## State

`Stateless.` — everything operates on the player's own fields, described in
[`sound_player.h`](sound_player.h.md).

## `set_sound_mask`

**Contract** — installs a new suppression mask and immediately retires every playing sound
that the new mask forbids. Setting the mask is therefore not a passive flag change; it has
the side effect of stopping sources. This is the intended behaviour: a creature entering a
state where it must be quiet becomes quiet on the same frame, not on its next update.

## `remove_active_sounds`

**Contract** — stops every sound in the given categories *without changing* the creature's
standing suppression mask. A one-shot "shut that up now".

```text
FUNCTION remove_active_sounds(mask)
  saved = sound_mask
  set_sound_mask(mask)      # retires everything matching `mask`
  set_sound_mask(saved)     # restores the standing policy; retires nothing new
```

**Notes** — the round-trip through `set_sound_mask` is the whole implementation, and it
works only because setting a mask retires as a side effect. It is a clever reuse in C++
and an obscure one to read; a rebuild should call the retirement pass directly with the
one-shot mask and leave the standing mask alone. The observable contract is what matters:
after this call, sounds in those categories are stopped and new ones in them are *not*
blocked.

The restore pass is not a no-op in principle — it re-applies the standing mask — but it
retires nothing extra, because anything the standing mask forbids was already retired the
last time it was set.

## `active_sound_count`

**Contract** — counts live sounds, with a switch for what "live" means. With the
playing-only flag, counts sources currently audible. Without it, also counts sounds that
are scheduled and whose start instant has arrived — that is, sounds the next update will
start. Linear in the queue.

**Notes** — the two meanings exist because callers ask two different questions: "am I
making noise" (audible only) versus "am I committed to making noise" (audible or due).
A sound still waiting out its random start delay counts as neither.

## `active_sound_type`

**Contract** — reports whether any live sound has *exactly* the given category mask.
Liveness uses the same "audible or due" test as the count above.

**Notes** — the comparison is equality of the whole mask, not intersection, unlike the
arbitration rule in [`sound_player.cpp`](sound_player.cpp.md) which tests intersection.
The asymmetry is real and load-bearing: arbitration asks "does this conflict with
anything", this query asks "is this exact kind of sound happening".

## `playing_sounds`, `objects`

**Contract** — expose the live-sound queue and the registered-kind table for inspection.
Read-only.

## `sound_prefix` (set / get)

**Contract** — set and read the creature's voice-folder prefix, which is prepended to
every authored name stem when a collection is built. The setter normalizes an absent value
to the empty string, so the prefix is never absent — concatenation sites can then be
unconditional.

**Notes** — because the prefix participates in the collection cache key, changing it
after kinds are registered does not reload them; it affects only kinds registered
afterwards. Set it before registration.

## `sound_collection.add`

**Contract** — loads one named sound file as an effect source with the given AI sound
type, returning a handle or nothing. Called only during collection construction.

**Notes** — the AI sound type travels with the load, not with playback: it is the
*perception* attribute the senses system reads when this sound reaches a listener, which
is why it is part of the collection's cache key and cannot be varied per instance.
