# src/xrCore/file_stream_reader.cpp

> The streaming reader over a whole loose file rather than a region of an archive: it opens and owns the file and its mapping.

**Needs** — [`file_stream_reader.h`](file_stream_reader.h.md) · [`stream_reader.h`](stream_reader.h.md) · [`stream_reader.cpp`](stream_reader.cpp.md) · [`xrDebug.h`](xrDebug.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`file_stream_reader.h`](file_stream_reader.h.md)
**Tier floor** — T1: it opens a file, creates a read-only mapping of it, and hands the mapping to the base reader.

## Purpose

The sliding-window reader is built to read *inside* an already-mapped archive and deliberately does not own anything. This variant is the case where the file stands alone: an unpacked level, a save game, anything read from a loose directory. It adds exactly one thing — ownership of the file handle and the mapping — and inherits the rest.

## State

```text
RECORD FileStreamReader EXTENDS StreamReader
  file : handle      # the open file; the base holds the mapping made from it
```

## `construct`

**Contract** — Takes a path and a window size. Opens the file for reading, shared for reading, failing if it does not exist. Queries its length and requires it to be non-zero. Creates a read-only mapping of the whole file and hands the base reader that mapping with a region covering the entire file. Blocks; allocates only the path conversion.

```text
FUNCTION construct(path, window_size) -> void
  file = open_for_read(normalized(path))       # separators converted to the platform's
  REQUIRE file is valid
  size = length_of(file)
  REQUIRE size > 0                             # a zero-length file cannot be mapped
  mapping = create_read_only_mapping(file)
  base.construct(mapping, start = 0, region = size, whole = size, window_size)
```

**Invariants** — Region length and whole-mapping length are the same here, which is what makes the base reader's end-of-window clamping degenerate into "clamp at end of file".

**Notes** — On the POSIX family the file descriptor doubles as the mapping handle, so there is one resource rather than two; on Windows they are separate and both must be closed. A rebuild has one handle or two depending on its platform layer, and the only decision that travels is that **this reader owns whatever it opened and the base reader owns none of it.**

## `destroy`

**Contract** — Unmaps the base reader's window first, then closes the mapping, then the file. The order matters: an unmapped region must not outlive its mapping.
