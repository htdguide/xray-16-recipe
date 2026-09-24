# src/utils/mp_gpprof_server/profiles_cache.h

> Declares the fixed-capacity, name-sorted cache of recently fetched profiles.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [`profile_request.h`](profile_request.h.md) · [`threads.h`](threads.h.md)

**Used by** — [`profiles_cache.cpp`](profiles_cache.cpp.md) · [`requests_processor.cpp`](requests_processor.cpp.md) · [`requests_processor.h`](requests_processor.h.md)

**Tier floor** — T2: capacity is computed from a byte budget and a record size, so the record's footprint is load-bearing.

## Purpose

Declares the surface implemented in [`profiles_cache.cpp`](profiles_cache.cpp.md). The
shape to notice is that an entry stores the player's name **inline** rather than by
reference, so an entry has a fixed size and the cache's capacity in entries is a memory
budget divided by that size. That is what makes the cache bounded without a separate
accounting pass.

The cache itself is not thread-safe; its caller holds a lock around every use.

## Exported units

- `profiles_cache` — the cache. Constructed with a memory budget in bytes.
- `search` — look a name up, expiring the entry if it is too old.
- `add` — insert or refresh an entry; reports whether the ordering was disturbed.
- `clear_expired` — drop entries that have aged out or were never read.
- `sort` — restore the name ordering that `search` depends on.
- `cache_item` — one entry: the name, the profile, when it was fetched, how often it has
  been served.
