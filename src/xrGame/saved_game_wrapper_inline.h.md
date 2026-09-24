# src/xrGame/saved_game_wrapper_inline.h

> The four accessors.

**Needs** — [`saved_game_wrapper.h`](saved_game_wrapper.h.md)
**Used by** — [`saved_game_wrapper.h`](saved_game_wrapper.h.md)
**Tier floor** — T3: field reads

## Purpose

Plain reads of the four facts extracted in
[`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md). No decisions.

## `game_time` · `level_id` · `level_name` · `actor_health`

**Contract** — return the in-world clock, the level identifier, the level display name and
the actor's normalized health as they were recovered at construction. All are valid after
construction, including after a failed read, which substitutes defaults rather than
signalling.
