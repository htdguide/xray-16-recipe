# src/xrGame/stalker_sound_data_inline.h

> Construction and emitter access for the stalker sound payload.

**Needs** — [`stalker_sound_data.h`](stalker_sound_data.h.md)
**Used by** — [`stalker_sound_data.h`](stalker_sound_data.h.md)
**Tier floor** — T3: field assignment behind two assertions.

## Purpose

Two one-line members kept out of the header for readability. A rebuild puts them on the type
and deletes this file. What survives is the pair of conditions they assert.

## `construct(emitter)`

**Contract** — a payload is *always* created with an emitter. There is no empty payload: a
sound either carries a stalker identity or carries no payload at all, which is why the
listener's visitor can assume an emitter exists the moment it is called.

## `object()`

**Contract** — the emitter, with the assertion that it is still linked. This is the
companion to `invalidate`: after severing, reading the emitter is a programming error rather
than a runtime condition the caller handles. The safe path is `accept`, which tests first.
