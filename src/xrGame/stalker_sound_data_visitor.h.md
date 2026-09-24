# src/xrGame/stalker_sound_data_visitor.h

> Declares the listener-side handler for a heard stalker.

**Needs** — [`stalker_sound_data_visitor.cpp`](stalker_sound_data_visitor.cpp.md) · [`sound_user_data_visitor.h`](sound_user_data_visitor.h.md) · [`stalker_sound_data_visitor_inline.h`](stalker_sound_data_visitor_inline.h.md)
**Used by** — [`stalker_sound_data_visitor.cpp`](stalker_sound_data_visitor.cpp.md) · [`stalker_sound_data_visitor_inline.h`](stalker_sound_data_visitor_inline.h.md)
**Tier floor** — T2: one case of the sound-payload visitor.

## Purpose

Declares the surface implemented in
[`stalker_sound_data_visitor.cpp`](stalker_sound_data_visitor.cpp.md). It fills the one case
of the sound-payload visitor that a stalker understands. A listener that meets a payload it
has no case for simply learns nothing, which is how the perception layer stays open to new
emitter kinds without every listener knowing about them.

## Exported units

- construction from the listening stalker — see
  [`stalker_sound_data_visitor_inline.h`](stalker_sound_data_visitor_inline.h.md).
- `visit(stalker payload)` — the propagation rule: inherit the speaker's danger, or acquire
  the speaker's enemy as a fictional sighting.
- `object()` — the listening stalker.
