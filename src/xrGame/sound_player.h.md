# src/xrGame/sound_player.h

> Declares the per-object sound player: a creature's catalogue of sound kinds, the priority and mutual-exclusion rules between them, and the queue of sounds currently scheduled or playing on its bones.

**Needs** — [`sound_player.cpp`](sound_player.cpp.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`sound_player_inline.h`](sound_player_inline.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`base_monster_feel.cpp`](ai/monsters/basemonster/base_monster_feel.cpp.md) · [`pseudodog.cpp`](ai/monsters/pseudodog/pseudodog.cpp.md) · [`rat_state_activation.cpp`](ai/monsters/rats/rat_state_activation.cpp.md) · [`zombie_state_attack_run_inline.h`](ai/monsters/zombie/zombie_state_attack_run_inline.h.md) · [`ai_trader.h`](ai/trader/ai_trader.h.md) · [`sound_collection_storage.cpp`](sound_collection_storage.cpp.md) · [`sound_collection_storage.h`](sound_collection_storage.h.md) · [`sound_player.cpp`](sound_player.cpp.md) · [`sound_player_inline.h`](sound_player_inline.h.md) · [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md) · [`stalker_animation_head.cpp`](stalker_animation_head.cpp.md) · [`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md) · _and 6 more_
**Tier floor** — T2: schedules and positions audio sources against a skeleton every frame

## Purpose

Declares the surface implemented in [`sound_player.cpp`](sound_player.cpp.md), plus the
record shapes the whole subsystem is built from. The records matter more than the methods
here, so they are given in full below; the contracts are in the implementation twin.

Exported units:

- `CSoundPlayer` — the per-object player. Built over a seeded random stream, so each
  creature's sound choices differ from its neighbours'.
- `add` / `remove` / `clear` — register, drop and drop-all sound *kinds*.
- `play` — schedule one sound of a kind, with a randomized start and stop delay.
- `update` — per-frame: retire finished sounds, start due ones, follow the bone.
- `reinit` / `reload` / `unload` — the entity lifecycle hooks.
- `set_sound_mask` / `remove_active_sounds` — suppress whole categories at once.
- `playing_sounds`, `active_sound_count`, `active_sound_type`, `need_bone_data`,
  `objects`, `sound_prefix` — queries, described in
  [`sound_player_inline.h`](sound_player_inline.h.md).

## State

```text
RECORD sound_params                    # how a sound kind behaves
  priority      : int                  # higher wins against a same-category rival
  synchro_mask  : int (32-bit, bitset) # categories this sound belongs to
  bone_name     : text                 # the skeleton bone it emits from

RECORD sound_collection_params         # what identifies a *set of files* on disk
  sound_prefix        : text           # comma-separated list of authored name stems
  sound_player_prefix : text           # per-creature path prefix, e.g. its voice folder
  max_count           : int            # how many numbered variants to probe for
  type                : enum ai_sound  # the AI-perception attributes of the emission
  # equality is field-wise over all four: this is the cache key in the shared storage

RECORD sound_collection_params_full : sound_params + sound_collection_params
  data : optional<user data>           # opaque payload the listener side reads back

RECORD sound_collection                # the loaded set, shared between creatures
  sounds        : list<sound handle>   # invariant: a kind with an empty list never plays
  last_sound_id : int                  # the previous draw, so the next avoids repeating it

RECORD sound_single                    # one scheduled or playing instance
  <sound_params copied from its kind>
  sound      : sound handle            # a clone; owned by this record
  start_time : int (world clock, ms)   # may be in the future: the sound is queued
  stop_time  : int (world clock, ms)   # start + clip length + a random tail
  started    : bool
  bone_id    : int                     # resolved once at schedule time; never BONE_NONE

RECORD sound_player
  sounds         : map<internal_type, (params_full, sound_collection)>
  playing_sounds : list<sound_single>
  sound_mask     : int (32-bit, bitset)  # categories currently suppressed
  object         : game object           # the emitter; supplies transform and skeleton
  sound_prefix   : text                  # invariant: never none; empty string instead
```

**Invariant across the records** — `bone_id` is resolved when a sound is scheduled, not
when it starts, because the kind's bone name is authored text and a missing bone is an
authoring error that must surface at the moment of the call, not silently at playback.
