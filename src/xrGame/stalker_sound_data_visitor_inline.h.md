# src/xrGame/stalker_sound_data_visitor_inline.h

> Construction and listener access for the stalker sound visitor.

**Needs** — [`stalker_sound_data_visitor.h`](stalker_sound_data_visitor.h.md)
**Used by** — [`stalker_sound_data_visitor.h`](stalker_sound_data_visitor.h.md)
**Tier floor** — T3: field assignment behind an assertion.

## Purpose

Two one-line members kept out of the header. A rebuild puts them on the type and deletes
this file.

## `construct(listener)` / `object()`

**Contract** — a visitor is always bound to a listening stalker, and the binding is never
cleared: unlike the payload, which can outlive its emitter, a visitor is owned by the
listener and dies with it. That asymmetry is the reason only one of the two types needs an
invalidation step.
