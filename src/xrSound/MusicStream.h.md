# src/xrSound/MusicStream.h

> Declares the dead music-stream table. Not compiled.

**Needs** — [`xr_streamsnd.h`](xr_streamsnd.h.md)
**Used by** — [`MusicStream.cpp`](MusicStream.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the table described in [`MusicStream.cpp`](MusicStream.cpp.md). Excluded from the build; a
rebuild implements neither file.

## Exported units

- **`MusicStreams`** — the slot table.
- **`create_sound` / `delete_sound`** — allocate a stream into a free slot, or release one.
- **`on_move` / `update` / `reload`** — the per-frame pump, the mark pass, and a reload stub that
  was never written.
