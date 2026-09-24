# src/xrCore/stream_reader.cpp

> A sliding memory-mapped window over a region of a file: reads arbitrarily large chunks of an archive without mapping the whole archive, and without copying it.

**Needs** — [`stream_reader.h`](stream_reader.h.md) · [`stream_reader_inline.h`](stream_reader_inline.h.md) · [`FS.h`](FS.h.md) · [`FS_impl.h`](FS_impl.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`xrstring.h`](xrstring.h.md) · [`xrMemory.h`](xrMemory.h.md)
**Used by** — [`file_stream_reader.cpp`](file_stream_reader.cpp.md) · [`stream_reader.h`](stream_reader.h.md) · [`stream_reader_inline.h`](stream_reader_inline.h.md)
**Tier floor** — T1: it maps file regions at allocation-granularity-aligned offsets and reads structures directly out of the mapping. A rebuild may instead buffer, but must then reproduce the same read semantics, and the whole reason this class exists is to *not* buffer.

## Purpose

Most files the engine reads are small enough to load whole. Some are not: a level's geometry and its spawn list run to hundreds of megabytes and live inside an archive that is larger still. This reader gives those a sequential-with-seeking interface over a window that slides, so the resident cost is one window regardless of file size, and the bytes are never copied out of the page cache.

It is separate from the whole-file reader because their failure modes differ: this one cannot hand out a pointer into the file that survives the next read.

## State

```text
RECORD StreamReader
  mapping           : handle          # the file mapping this reader draws from
  start_offset      : int             # where this reader's region begins inside the mapping
  file_size         : int             # length of the region; the reader's notion of "the file"
  archive_size      : int             # length of the whole mapping; a window may not run past it
  window_size       : int             # requested window length

  offset_from_start : int             # logical position of the window's first usable byte
  current_window    : int             # usable bytes in the window, from start_pointer
  mapped_base       : address         # what was actually mapped; granularity-aligned
  start_pointer     : address         # first usable byte = mapped_base + alignment slack
  current_pointer   : address         # the cursor
```

**Invariants**

- `start_pointer <= current_pointer <= start_pointer + current_window`. Every operation asserts it.
- The logical position is `offset_from_start + (current_pointer - start_pointer)` — there is no separate cursor field, and seeking is expressed as a relative advance from the current position.
- **Mappings must begin at a multiple of the platform's allocation granularity.** The requested start almost never is, so the reader maps from the granularity boundary below it and remembers the slack; `start_pointer` is past that slack and `current_window` excludes it.
- The end is rounded *up* to a granularity boundary and then clamped to the mapping's length, so a window near the end of the archive is short.
- The effective window is at least the allocation granularity, whatever the caller asked for.

## `construct` / `destroy`

**Contract** — Construction takes a file mapping, the region's start and length within it, the whole mapping's length, and a requested window size; it maps the first window immediately. Destruction unmaps. Neither owns the mapping — see [`file_stream_reader.cpp`](file_stream_reader.cpp.md) for the variant that does.

## `map`

**Contract** — Replaces the window so that a given logical offset is the first usable byte. Unmaps nothing — the caller is expected to have unmapped, which is what the remap helper does.

```text
FUNCTION map(new_offset) -> void
  REQUIRE new_offset <= file_size
  offset_from_start = new_offset

  wanted      = start_offset + new_offset
  aligned_start = wanted rounded DOWN to a granularity boundary
  wanted_end  = wanted + window_size
  aligned_end = wanted_end rounded UP to a granularity boundary
  IF aligned_end > archive_size THEN aligned_end = archive_size   # short final window

  mapped_base     = map_region(mapping, aligned_start, aligned_end - aligned_start)
  slack           = wanted - aligned_start
  start_pointer   = mapped_base + slack
  current_pointer = start_pointer
  current_window  = (aligned_end - aligned_start) - slack
```

## `advance`

**Contract** — Moves the cursor by a signed amount. Remaps when the move leaves the window in either direction; otherwise it is a pointer adjustment. Seeking to an absolute position is expressed as advancing by the difference.

```text
FUNCTION advance(delta) -> void
  inside = current_pointer - start_pointer
  IF inside + delta >= current_window OR inside + delta < 0 THEN
    remap(offset_from_start + inside + delta)   # unmap, then map at the new offset
  ELSE
    current_pointer = current_pointer + delta
```

**Notes** — Advancing to exactly the window's end remaps rather than parking at the boundary. That keeps the invariant "the cursor always has at least one readable byte ahead of it unless the file is exhausted", which the string reader below depends on.

## `read`

**Contract** — Copies a requested number of bytes into a caller-supplied buffer, crossing as many windows as needed, and leaves the cursor after them. Blocks on page faults, allocates nothing.

```text
FUNCTION read(buffer, count) -> void
  inside = current_pointer - start_pointer
  IF inside + count < current_window THEN         # the common case: one window
    copy count bytes; advance cursor; RETURN

  available = current_window - inside
  WHILE current_window < remaining count
    copy `available` bytes into buffer
    advance buffer and decrease count
    advance(available)                            # forces a remap
    available = current_window                    # a fresh window is wholly available
  copy the remaining count
  advance(count)
```

## `find_chunk` and `open_chunk`

**Contract** — The chunked container format is `(identifier, size, payload)` repeated, and a chunk's payload may itself be a chunk sequence. Finding a chunk positions the cursor at its payload and reports the payload's length; opening one returns a *new* reader bounded to that payload, so nested chunks are read through independent cursors.

The identifier's top bit is a **compression mark**: the search masks it off when comparing and reports it separately.

```text
FUNCTION open_chunk(id) -> optional<StreamReader>
  size = find_chunk(id)                 # positions the cursor at the payload
  IF size == 0 THEN RETURN none
  REQUIRE the chunk is not marked compressed
        # a compressed chunk must be decompressed into memory whole;
        # a sliding window over it would have to decompress on every remap
  RETURN new StreamReader over (this mapping,
                                start_offset + current position,
                                size, archive_size, window_size)
```

**Notes** — The refusal to stream a compressed chunk is the important line. It is why the level files whose chunks are large are stored uncompressed in the shipped archives, and why a rebuild that compresses them breaks streaming rather than merely slowing it.

## `read_string`

**Contract** — Reads a null-terminated string and interns it. Two cases, and the split is the whole algorithm:

```text
FUNCTION read_string() -> InternedString
  LOOP
    scan forward from the cursor for a terminator, up to the window's end

    IF a terminator was found AND we have not yet copied anything THEN
      # Fast path: the string lies entirely inside the current window, so
      # it can be interned straight out of the mapping with no copy at all.
      result = intern(bytes at the cursor)
      cursor = just past the terminator
      RETURN result

    # Slow path: the string straddles a window boundary. Copy what this
    # window holds into scratch, remap so the window starts exactly where
    # we stopped, and continue.
    IF this is the first straddle THEN scratch = 4096 bytes of stack space
    copy the rest of the window into scratch
    REQUIRE the accumulated length stays within 4096
    remap(offset_from_start + what we just copied)
  UNTIL the last copied byte was a terminator
  RETURN intern(scratch)
```

**Invariants** — A string in this format may not exceed 4096 bytes. That is a hard limit of the reader, asserted, not a buffer that grows.

**Notes** — The fast path is the point of the whole class: names in level data are short and dense, and interning directly from the mapping means a level's tens of thousands of names cost no copies at all.

## `elapsed` / `length` / `tell` / `seek` / `at_end` / `close`

**Contract** — Position and length queries derived from the fields above; `at_end` is "nothing elapsed". Closing destroys the mapping window and then releases the reader itself — the reader owns its own lifetime, which is how callers of `open_chunk` release the readers they were handed.
