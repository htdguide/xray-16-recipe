# src/xrGame/smart_cover_storage.h

> Declares the process-wide cache of smart-cover *descriptions*, keyed by the configuration table that defines them.

**Needs** — [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_storage.cpp`](smart_cover_storage.cpp.md)
**Used by** — [`cover_manager.cpp`](cover_manager.cpp.md) · [`smart_cover.cpp`](smart_cover.cpp.md) · [`smart_cover_storage.cpp`](smart_cover_storage.cpp.md)
**Tier floor** — T2: a reference-counted cache with a deferred-eviction policy

## Purpose

Declares the surface implemented in
[`smart_cover_storage.cpp`](smart_cover_storage.cpp.md). One constant lives here and is
load-bearing: a released description is kept for **300 seconds** before it is actually
freed, so that a smart cover which unloads and reloads within that window pays no parse
cost.

Exported units:

- `description(table_id)` — get or build the description for a named configuration table.
- `collect_garbage()` — drop descriptions whose grace period has expired.
- destructor — assert everything was released, then free.

## State

```text
RECORD storage
  descriptions : list<description>   # each reference-counted; held until grace expires
  GRACE_PERIOD : int = 300_000 ms    # see the .cpp twin for why it is this long
```
