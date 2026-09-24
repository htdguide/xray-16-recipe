# src/xrGame/stalker_sound_data.h

> Declares the stalker's sound payload.

**Needs** — [`stalker_sound_data.cpp`](stalker_sound_data.cpp.md) · [`stalker_sound_data_inline.h`](stalker_sound_data_inline.h.md)
**Used by** — [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`stalker_sound_data.cpp`](stalker_sound_data.cpp.md) · [`stalker_sound_data_inline.h`](stalker_sound_data_inline.h.md) · [`stalker_sound_data_visitor.cpp`](stalker_sound_data_visitor.cpp.md)
**Tier floor** — T2: one typed payload over the sound layer's payload interface.

## Purpose

Declares the surface implemented in
[`stalker_sound_data.cpp`](stalker_sound_data.cpp.md). It fills the sound layer's
user-payload slot: a sound may carry one, and a listener discovers what kind it is by
offering it a visitor rather than by testing its type.

## Exported units

- construction from the emitting stalker — see
  [`stalker_sound_data_inline.h`](stalker_sound_data_inline.h.md).
- `invalidate()` — sever the emitter link when the emitter dies.
- `accept(visitor)` — offer this payload to a listener's visitor, unless the emitter is gone.
- `object()` — the emitting stalker.
