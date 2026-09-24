# src/xrGame/stalker_animation_data_storage_inline.h

> The process-wide animation-table cache, created the first time anything asks for it.

**Needs** — [`stalker_animation_data_storage.h`](stalker_animation_data_storage.h.md)
**Used by** — [`stalker_animation_data_storage.cpp`](stalker_animation_data_storage.cpp.md) · [`stalker_animation_data_storage.h`](stalker_animation_data_storage.h.md)
**Tier floor** — T3: a lazily created singleton

## Purpose

One definition: the accessor for the single shared cache.

## The storage accessor

**Contract** — return the process-wide animation-table cache, creating it on first call.
Never returns nothing. Not thread-safe; every caller is on the game thread.

**Notes** — created on demand rather than at startup because it is only reached when the
first stalker is brought online, and a game session with no stalkers — a menu, a
multiplayer map without them — should not pay for it. It is never destroyed on the way out;
its contents are emptied on level unload instead, which is what actually matters because the
tables reference motion banks that unload with the level.
