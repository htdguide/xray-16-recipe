# src/xrCore/stream_reader_inline.h

> The trivial half of the streaming reader: position arithmetic, the unmap/remap pair, and self-release.

**Needs** — [`stream_reader.h`](stream_reader.h.md) · [`stream_reader.cpp`](stream_reader.cpp.md)
**Used by** — [`stream_reader.cpp`](stream_reader.cpp.md) · [`stream_reader.h`](stream_reader.h.md)
**Tier floor** — T1: it unmaps a region by base address and length.

## Purpose

Carries the parts of the streaming reader that are arithmetic over its fields rather than algorithm. The split from [`stream_reader.cpp`](stream_reader.cpp.md) is a compilation concern — these are called in inner loops and are wanted inline — and is arbitrary from a rebuild's point of view: merge them.

## Exported units

- **Unmap** — releases the current window by its *mapped base and full length*, not by the usable pointer and usable length. The alignment slack is part of the mapping and must be given back with it; unmapping from `start_pointer` leaks the slack page on every remap.
- **Remap** — unmap, then map at a new logical offset. The only correct way to move the window.
- **Tell** — `offset_from_start + (cursor - start_pointer)`. There is no stored cursor position.
- **Seek** — advance by `target - tell()`.
- **Elapsed** — `file_size - tell()`; the "at end" test is this being non-positive.
- **Length** — the region's length, which is what this reader calls "the file".
- **Mapping handle accessor**.
- **Close** — destroy the window and then release the reader object itself. Readers handed out by chunk-opening are freed this way, so the caller never sees the allocation.
