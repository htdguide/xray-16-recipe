# src/xrSound/SoundRender_Core_SourceManager.cpp

> The shared-source cache: one decoded-and-described asset per path, for the life of the process.

**Needs** — [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a keyed cache with a lock.

## Purpose

A hundred emitters may play the same footstep asset. Each needs the asset's *description* — sample
rate, channel count, total length, attenuation distances, the AI attributes — but none needs its
own copy, and each opens its own decoder over the same file anyway. So sources are shared and
identified by path.

## `get_or_load_source`

**Contract** — Returns the shared source for an asset path, loading it on first request. The key is
the path lowercased with its extension stripped, which is what makes `weapons/ak74_shoot`,
`Weapons/AK74_Shoot.ogg` and `weapons/ak74_shoot.wav` the same asset — the game data references
sounds with inconsistent case and with a stale `.wav` extension left over from before the engine
moved to Vorbis. Returns nothing if the asset cannot be loaded. Safe to call from the streaming
worker as well as the update thread.

```text
FUNCTION get_or_load_source(path) -> optional<Source>
  key ← lowercase(strip_extension(path))
  LOCK cache DURING
    IF key IN sources THEN RETURN sources[key]

  candidate ← load_source(key)          # file I/O, outside the lock
  IF candidate failed THEN RETURN none

  LOCK cache DURING
    sources[key] ← candidate
    RETURN candidate
```

**Notes** — The load happens outside the lock, so two threads can race to load the same asset and
the second one's work is wasted. That is the deliberate trade: a sound asset's load is a header
parse and a metadata read, not a decode, so duplicating it costs microseconds, whereas holding the
cache lock across file I/O would stall every other emitter's cursor swap. A rebuild that would
rather not waste the work should insert a per-key in-flight marker, not widen the lock.

The second insertion overwrites rather than checking for a racing winner, which leaks the loser's
description. This is a genuine small bug, not a decision; a rebuild should keep the first entry.

## `release_source`

**Contract** — Does nothing. Sources are never evicted.

**Notes** — This is not an oversight, it is the cache policy: a level's entire sound bank is
descriptions, not audio data — audio is streamed from disk on demand — so the cache's footprint is
a few hundred small records and there is nothing to reclaim. The function exists so that call sites
read symmetrically and so that a rebuild that *does* want eviction has one place to put it. The
only bulk teardown is at shutdown, in [`SoundRender_Core.cpp`](SoundRender_Core.cpp.md).
