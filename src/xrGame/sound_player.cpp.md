# src/xrGame/sound_player.cpp

> The per-creature sound scheduler: it owns which sound kinds a creature knows, arbitrates between kinds that must not overlap, delays and staggers playback randomly, and keeps every live source glued to the bone it came from.

**Needs** — [`sound_player.h`](sound_player.h.md) · [`sound_collection_storage.h`](sound_collection_storage.h.md) · [`ai_space.h`](ai_space.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`sound_player.h`](sound_player.h.md)
**Tier floor** — T2: per-frame source repositioning against a skeleton pose; allocation per playing instance

## Purpose

Creatures in this game talk, breathe, grunt when hit and scuff their feet, and the rules
governing all of that are the same rules. This file is those rules. It sits between the
behaviour layer — which says "make the *attack* sound" — and the audio device, which
wants a positioned source and a PCM stream.

Four decisions live here and nowhere else:

1. **A sound *kind* is a set, not a file.** Asking to play a kind draws one of several
   authored variants, avoiding the one drawn last time.
2. **Kinds are mutually exclusive by category, and priority breaks ties.** A creature does
   not shout while it is already shouting.
3. **Playback is deliberately not immediate.** A requested sound gets a random start delay
   and a random silent tail, so that a squad of creatures issued the same order does not
   emit one synchronized chorus.
4. **A sound is attached to a bone, and follows it.** Emission position is recomputed from
   the animated pose every frame while the source is alive.

## State

The records are in [`sound_player.h`](sound_player.h.md). What this file adds is the
lifecycle invariant tying them together:

```text
# A sound_single owns its cloned handle and must be explicitly torn down. The only
# code permitted to drop one is the retirement pass, which stops the source first.
# invariant: no sound_single outlives the player
# invariant: playing_sounds is empty across reload and across unload
```

## `add`

**Contract** — registers a sound kind under a caller-chosen `internal_type` tag, giving it
a file-name prefix, a variant count, an AI sound type, a priority, a category mask, an
emitting bone name and an optional user-data payload. Returns how many variants were
actually found on disk; returns zero and changes nothing if the tag is already registered.
The variant set itself is not loaded here — it is fetched from the process-wide collection
storage, so several creatures with the same voice share one set of loaded sounds.

```text
FUNCTION add(prefix, max_count, type, priority, mask, internal_type, bone_name, data) -> int
  IF internal_type ALREADY IN sounds
    RETURN 0                          # first registration wins; re-registering is a no-op
  params = sound_collection_params_full{
      priority, mask, bone_name,
      sound_prefix        = prefix,
      sound_player_prefix = this.sound_prefix,   # the creature's own voice folder
      max_count, type, data }
  collection = shared_collection_storage.get(params)   # cached by value of params
  sounds[internal_type] = (params, collection)
  RETURN size of collection.sounds
```

**Notes** — the caller's tag is an arbitrary small integer from the behaviour layer's own
enumeration, not anything the data supplies. Returning the found-variant count lets the
caller detect "this creature has no attack sounds" at registration rather than at the
first attack.

## `play`

**Contract** — schedules one sound of the given kind. Silently does nothing if the kind is
unknown, if the kind is currently forbidden, or if its variant set is empty. Otherwise it
clones a variant, resolves the bone, computes a start and a stop instant, appends the
instance to the playing list, and — only if the start instant has already passed — starts
it immediately. Allocates one sound handle. Hard-fails if the named bone does not exist on
the creature's skeleton.

```text
FUNCTION play(internal_type, max_start, min_start, max_stop, min_stop, id = none)
  IF NOT permitted(internal_type)        RETURN     # see `permitted` below
  (params, collection) = sounds[internal_type]
  IF collection.sounds IS empty          RETURN

  retire_conflicting(params.synchro_mask)  # the new sound displaces its own category

  s = new sound_single FROM params
  s.bone_id = skeleton bone id FOR params.bone_name
  REQUIRE s.bone_id EXISTS   ELSE FAIL WITH "sound bone missing"
  s.sound = clone of collection.pick(id)   # see sound_collection.pick
  s.sound.emitter   = this.object          # so the listener can attribute the sound
  s.sound.user_data = params.data          # AI-perception payload

  REQUIRE max_start >= min_start AND max_stop >= min_stop
  delay = 0
  IF max_start > 0
    delay = max_start > min_start ? min_start + random(max_start - min_start) : max_start
  s.start_time = now + delay

  tail = 0
  IF max_stop > 0
    tail = max_stop > min_stop ? min_stop + random(max_stop - min_stop) : max_stop
  s.stop_time = s.start_time + clip_length_ms(s.sound) + tail

  APPEND s TO playing_sounds
  IF now >= s.start_time
    start s AT bone_position(s)
```

**Invariants** — `stop_time` is always at least `start_time` plus the clip's own length,
so the retirement test can never cut a sound off mid-play; the tail only extends the
window during which the *kind* remains considered busy.

**Notes** — the two randomized windows are not decoration. `min_start`/`max_start`
staggers a group; `min_stop`/`max_stop` inserts silence *after* the clip, which is what
stops a repeating idle sound from becoming metronomic. A rebuild that plays immediately
with no tail will sound mechanically wrong on the shipped data even though every file is
correct.

Calling `retire_conflicting` with the *new* sound's mask, before inserting it, is what
gives the new request precedence over an already-playing sound of the same category. The
priority comparison in `permitted` has already decided that this is allowed.

## `permitted` (the arbitration rule, `check_sound_legacy` in the source)

**Contract** — answers whether a kind may start right now. Three reasons for no: the tag
is not registered; the kind's categories intersect the player's current suppression mask;
or some already-playing sound shares a category with it and has priority at least as
strong. Pure query.

```text
FUNCTION permitted(internal_type) -> bool
  IF internal_type NOT IN sounds          RETURN false
  candidate = sounds[internal_type].params
  IF candidate.synchro_mask INTERSECTS sound_mask   RETURN false
  FOR EACH s IN playing_sounds
    IF s.synchro_mask INTERSECTS candidate.synchro_mask
      IF s.priority <= candidate.priority
        RETURN false
  RETURN true
```

**Notes** — the comparison reads backwards on first sight and is worth stating plainly:
**a numerically lower `priority` value is the stronger claim.** An incumbent whose value
is less than or equal to the candidate's blocks it; only an incumbent with a strictly
larger value can be displaced. The authored data assigns small numbers to speech and large
ones to ambience, so speech interrupts breathing and never the reverse.

The category mask is a bitset, so one kind can belong to several mutually exclusive groups
at once — a sound can be "vocal" and "combat" and conflict with either family.

## `update`

**Contract** — called once per frame with the frame delta (which it does not use; the
world clock drives everything). Retires finished and forbidden sounds, then advances the
survivors. Allocates nothing.

```text
FUNCTION update(delta)
  retire_conflicting(sound_mask)     # forbidden categories and expired instances
  FOR EACH s IN playing_sounds
    IF s IS audible
      move s TO bone_position(s)     # follow the animated bone
    ELSE IF NOT s.started AND now >= s.start_time
      start s AT bone_position(s)
```

**Notes** — retirement runs *before* the advance, so a sound suppressed this frame never
gets one more position update. The frame delta being ignored is deliberate: sound timing
is on the world clock, which is the same clock the alife and scheduler use, so a paused or
fast-forwarded simulation moves sound with it.

## `retire_conflicting` (`remove_inappropriate_sounds`)

**Contract** — removes every playing instance that either belongs to a suppressed category
or has finished. Each removal stops the source if it is still audible and releases the
handle. Takes a mask rather than reading the player's own, because `play` uses it to clear
a rival category before starting.

```text
FUNCTION retire_conflicting(mask)
  REMOVE s FROM playing_sounds WHERE
      (s.synchro_mask INTERSECTS mask)
      OR (s IS NOT audible AND s.stop_time <= now)
  # each removal: stop the source if audible, then release the handle
```

**Invariants** — "not audible and past its stop time" is the finished test, and both
halves are needed. A queued sound whose start time has not arrived is not audible but is
also not past its stop time, so it survives; a long clip still playing past its nominal
stop time is audible, so it also survives and is allowed to finish.

## `bone_position` (`compute_sound_point`)

**Contract** — returns the world position a sound emits from: the creature's transform
composed with the current pose transform of the sound's bone. Recomputed per frame per
live source. Requires the creature to have an animated visual.

**Notes** — this is why `need_bone_data` exists: computing it forces the animation system
to have an up-to-date pose, which is expensive, so the sound player is asked first whether
any sound needs it this frame.

## `need_bone_data`

**Contract** — reports whether any playing instance is audible or is due to start this
frame; that is, whether `update` will need a current skeleton pose. Pure query, used by
the creature's update to decide whether to force a pose evaluation.

## `clear`, `reinit`, `reload`, `unload`

**Contract** — the lifecycle hooks, in the order the entity lifecycle calls them.

- `clear` — forget every registered kind, tear down every playing instance, reset the
  suppression mask to "nothing suppressed".
- `reinit` — nothing. It exists to satisfy the lifecycle shape; a rebuild may drop it.
- `reload(section)` — asserts nothing is playing, then clears and resets the voice prefix.
  The assertion encodes a real ordering rule: **a creature must be silent before it is
  reconfigured**, because a reload swaps the collections out from under any live source.
- `unload` — suppress *all* categories, which retires everything, then assert silence.

**Notes** — `unload` suppressing everything rather than calling `clear` is the correct
shape: it routes teardown through the same retirement path that stops sources properly,
instead of a second destruction path that could diverge from it.

## `sound_collection` (construction: finding the variants on disk)

**Contract** — builds the shared variant set for one collection key. The name stem is a
comma-separated *list* of stems; for each stem it probes the virtual filesystem for a bare
file and then for numbered variants zero through the count, loading each that exists.
Missing files are not errors — the probe is how the engine discovers how many variants an
artist actually authored. An empty result is legal and logged in checked builds; kinds
with empty sets simply never play.

```text
FUNCTION build_collection(params) -> sound_collection
  FOR EACH stem IN split(params.sound_prefix, ",")
    base = params.sound_player_prefix + stem
    IF sound file exists AT "$game_sounds$/" + base
      APPEND loaded sound TO sounds
    FOR i IN 0 .. params.max_count - 1
      IF sound file exists AT "$game_sounds$/" + base + i
        APPEND loaded sound TO sounds
```

**Notes** — the unnumbered file and the numbered ones are both accepted, and both may be
present, because the shipped data uses both conventions. The probe-until-absent shape is a
**data discovery mechanism, not a fallback**: there is no manifest of how many variants
exist, so a rebuild must probe too or it will silently load a subset of the shipped audio.

The set holds loaded sound handles that are *cloned* per playing instance, never played
directly, so the shared set is never mutated by a creature using it. Teardown asserts no
handle in the set is still audible, which catches a clone that escaped its owner.

## `sound_collection.pick` (`random`)

**Contract** — returns one variant. With an explicit valid index, returns that one. With
no index, returns a random variant **that is not the one returned last time**, unless the
set holds two or fewer, in which case a plain uniform draw is used. Records the choice.

```text
FUNCTION pick(id) -> sound
  IF id IS given AND id < count(sounds)
    last = id ;  RETURN sounds[id]
  IF count(sounds) <= 2
    last = random(count) ;  RETURN sounds[last]
  REPEAT
    r = random(count)
  UNTIL r != last
  last = r ;  RETURN sounds[r]
```

**Notes** — the "two or fewer" special case is not an optimisation, it is a termination
guard: with one variant the rejection loop would never end, and with two it would
alternate deterministically, which is a worse artefact than an occasional repeat. Above
two, forbidding an immediate repeat is what makes a handful of grunt samples sound like
variety instead of a loop.

The random stream is seeded per creature and per collection from the cycle counter, so two
creatures constructed in the same frame still diverge.
