# src/xrSound/MusicStream.cpp

> Dead code: a slot table over the pre-Vorbis music streamers. Not compiled.

**Needs** — [`MusicStream.h`](MusicStream.h.md) · [`xr_streamsnd.h`](xr_streamsnd.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a list with slot reuse.

## Purpose

Held the set of active music streams and pumped them once per frame. Excluded from the build; its
job is now done by ordinary emitters in a scene, which is strictly better because music then
participates in the same voice ranking, pausing and volume model as everything else.

**A rebuild should not implement this file.**

## What it decided

One thing, and it is a familiar pattern rather than a domain decision: a stream's slot in the table
is reused rather than compacted, so that a caller holding an index keeps it valid across
creations and deletions. The live code does not need this because callers hold handles, not
indices.

Its update did not advance the streams; it only marked each as needing an update, leaving the work
to a second pass. That split — mark in one pass, do the work in another — survives in the live
chapter as the update/render split, for the same reason: the marking pass may run when rendering
does not.

## Notes

Deletion clears the slot's entry but the surrounding code never nulls it, so the slot-reuse scan
never finds a free slot. The table only grows. This is a bug in dead code; noted so a reader does
not try to preserve it.
