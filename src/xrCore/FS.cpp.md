# src/xrCore/FS.cpp

> The chunked binary container: how every engine file that is not text is framed, read, written, and compressed.

**Needs** — [`FS.h`](FS.h.md) · [`FS_internal.h`](FS_internal.h.md) · [`FS_impl.h`](FS_impl.h.md) · [`lzhuf.h`](lzhuf.h.md) · [`LocatorAPI.h`](LocatorAPI.h.md) · [`xrMemory.h`](xrMemory.h.md) · [`_compressed_normal.h`](_compressed_normal.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`FS.h`](FS.h.md) · [`FS_internal.h`](FS_internal.h.md)
**Tier floor** — T1: readers hand out a pointer *into* a memory-mapped region and callers cast structures over it in place; the mapping must survive as long as the reader.

## Purpose

Every binary asset the engine reads — level geometry, models, animation banks, spawn files, save games — is a tree of chunks. This file defines the chunk framing and the two halves that traverse it: a **writer** that appends and back-patches chunk sizes, and a **reader** that finds a chunk by identifier and hands back a sub-reader scoped to its payload. It also defines the four kinds of reader the rest of the engine sees (over a private buffer, over a borrowed buffer, over a memory-mapped file, over a decompressed blob) and the whole-file helpers that sit outside the virtual filesystem.

The split from `LocatorAPI.cpp` is real and worth keeping: this file knows nothing about logical roots, archives or path resolution. It is handed bytes.

## State

The chunk format is **frozen**. Everything in the game data is framed this way.

```text
RECORD Chunk                     # on disk, little-endian, no padding
  id     : int (32-bit)          # high bit (0x80000000) is the compression mark,
                                 # not part of the identity
  size   : int (32-bit)          # payload bytes, NOT counting these 8 header bytes
  payload: bytes                 # `size` bytes; may itself be a sequence of Chunks

# invariant: chunks are laid end to end with no alignment padding between them;
#   a reader positioned at a chunk boundary can always read 8 bytes or is at EOF.
# invariant: when the compression mark is set, `size` is the COMPRESSED length,
#   and the payload is an LZ-Huffman stream whose decompressed length is
#   recovered from the stream itself, not from the header.
# invariant: chunk identifiers are unique only by convention — find-by-id
#   returns the FIRST match in file order. Several files rely on that.
```

```text
RECORD Reader                    # a cursor over a contiguous byte range
  data     : bytes               # borrowed or owned, depending on the flavour
  position : int                 # 0 .. length
  length   : int
  iter_pos : int                 # absolute offset in the PARENT at which this
                                 # chunk's payload ends — how the iterator
                                 # resumes after a sub-reader is closed
  last_hit : int                 # search accelerator, see `find_chunk`
```

```text
RECORD Writer
  name       : text              # target path; also the key the filesystem
                                 # re-registers the file under when closed
  open_chunks: list<int>         # stack of positions of each open chunk's
                                 # size field, for back-patching
# invariant: the stack must be empty when the writer is destroyed. A non-empty
#   stack means a chunk was opened and never closed, which silently corrupts
#   the file — this is checked and is fatal.
```

## `open_chunk` / `close_chunk` / `w_chunk` (writer side)

**Contract** — `open_chunk(id)` appends the identifier, remembers where the size field will live, and writes a placeholder zero. `close_chunk` seeks back to the remembered position, writes the byte count between there and the current end (excluding the four bytes of the size field itself), and seeks forward again. Chunks nest: the stack makes a chunk opened inside another chunk patch the correct slot. `chunk_size` reports the payload length of the innermost open chunk without closing it.

**Invariants** — every `open_chunk` is matched by exactly one `close_chunk` before the writer is finalized; the writer must support seeking backwards, which is why a writer cannot be a pure append-only stream.

```text
FUNCTION open_chunk(w, id)
  write_u32(w, id)
  PUSH tell(w) ONTO w.open_chunks
  write_u32(w, 0)                # placeholder

FUNCTION close_chunk(w)
  end   := tell(w)
  slot  := POP w.open_chunks
  seek(w, slot)
  write_u32(w, end - slot - 4)   # payload length excludes the size field
  seek(w, end)

FUNCTION w_chunk(w, id, data)
  open_chunk(w, id)
  IF id HAS compression_mark THEN write_compressed(w, data)
  ELSE write(w, data)
  close_chunk(w)
```

**Notes** — the compression mark lives in the identifier, so a reader that ignores compression still parses the container correctly: it skips a chunk by its stored size whether or not the payload is compressed. That is the reason the mark is a bit of the id rather than a separate field.

## `find_chunk`

**Contract** — given a chunk identifier, position the reader at the start of that chunk's payload and return the payload length; return zero and leave the position unspecified if no such chunk exists. Optionally reports whether the payload carries the compression mark. Searching is a linear walk from the start, skipping each non-matching chunk by its stored size.

**Invariants** — comparison ignores the compression mark, so `find_chunk(3)` matches a chunk written as `3 | compression_mark`. The reported payload must fit inside the reader's range; a chunk claiming more bytes than remain is a corrupt file.

```text
FUNCTION find_chunk(r, id) -> (size: int, compressed: bool)
  # Accelerator: chunks are almost always read in the order they were written,
  # so try the position just past the previously found chunk first. One probe.
  IF r.last_hit IS NOT 0 THEN
    seek(r, r.last_hit)
    type := read_u32(r); size := read_u32(r)
    IF (type WITHOUT compression_mark) = id THEN GOTO found

  rewind(r)
  WHILE NOT at_end(r)
    type := read_u32(r); size := read_u32(r)
    IF (type WITHOUT compression_mark) = id THEN GOTO found
    advance(r, size)
  r.last_hit := 0
  RETURN (0, false)

found:
  # remember where this chunk ends, so the next lookup is one probe
  IF tell(r) + size < r.length THEN r.last_hit := tell(r) + size
  ELSE r.last_hit := 0
  RETURN (size, type HAS compression_mark)
```

**Notes** — the accelerator is worth naming because it changes the cost of the common access pattern from quadratic to linear. A model file has a few dozen chunks and is read in written order; without the probe, loading a level's visuals re-walks every chunk list for every field. Three other search strategies were tried (a full pre-scan into a sorted vector, a lazily-populated hash map, and a plain linear scan) and the single-probe heuristic won; a rebuild need only reproduce the *behaviour*, and may choose any index.

## `open_chunk` (reader side)

**Contract** — find a chunk by identifier and return a new reader scoped to its payload, or nothing if absent. If the chunk is compressed, decompress it into a fresh buffer and return a reader that owns and frees that buffer; otherwise return a reader that borrows the parent's bytes with no copy. The returned reader records the absolute end of the chunk in the parent so an iteration can resume there. The caller closes the sub-reader; closing it must not invalidate the parent.

**Notes** — the borrowing case is the reason this layer is T1: the sub-reader is a pointer into the parent's mapping, and the parent — which may be a memory-mapped archive region — must outlive it. A rebuild that copies instead pays a level-load-sized memcpy.

## `open_chunk_iterator`

**Contract** — walk the chunks of a container in file order, one per call, returning the identifier and a reader over the payload. Passing nothing starts at the beginning; passing the previous sub-reader seeks to where that chunk ended, closes it, and opens the next. Returns nothing when fewer than eight bytes remain. As with the by-id form, a compressed payload is decompressed into an owned buffer.

```text
FUNCTION next_chunk(parent, previous) -> optional<(id, Reader)>
  IF previous IS none THEN rewind(parent)
  ELSE
    seek(parent, previous.iter_pos)   # absolute end of the previous chunk
    close(previous)                   # the iterator owns the sub-readers
  IF remaining(parent) < 8 THEN RETURN none
  id   := read_u32(parent)
  size := read_u32(parent)
  ...                                 # same branch as open_chunk
```

**Notes** — "iterate the children" is how variable-length lists are stored: a container chunk whose payload is a run of chunks numbered 0, 1, 2… Several formats use the *ordinal* of the chunk as the array index, so iteration order is load-bearing, not cosmetic.

## Reader primitives — the wire vocabulary

**Contract** — a reader exposes fixed-width little-endian scalar reads (8/16/32/64-bit signed and unsigned, and 32-bit float), whole-struct reads for the vector and colour types, two string conventions, and the quantized forms. Every read advances the cursor by exactly the width consumed. Reading past the end is a programming error, not a recoverable one.

**Invariants** — no byte-swapping happens anywhere; the formats are little-endian memory images (see [platform assumptions](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions)). A big-endian rebuild must swap at every one of these call sites.

Two string conventions coexist and are not interchangeable:

```text
FUNCTION read_line(r) -> text        # "r_string": for text-ish payloads
  # Consumes up to the first carriage-return or line-feed, then consumes the
  # whole run of terminators, so CRLF and bare LF both advance exactly one line.
  # The returned text excludes the terminators.

FUNCTION read_cstring(r) -> text     # "r_stringZ": for structured payloads
  # Consumes up to and including a single zero byte. The zero is consumed;
  # the text excludes it.
```

**Notes** — the writer's matching `write_line` emits carriage-return *then* line-feed unconditionally, on every platform. That is frozen: files written on one platform are read on another.

When a string is read into an interned string, the reader may take the length from the interned result rather than re-scanning — the interning layer already knows it. That is an optimization, not a format fact.

## Quantized reads and writes

**Contract** — several formats store a bounded real in 16 or 8 bits to save space. These are *lossy and frozen*: the exact mapping must be reproduced or shipped data shifts.

```text
FUNCTION quantize_16(value, min, max) -> int (16-bit)
  q := (value - min) / (max - min)          # value must lie in [min, max]
  RETURN floor(q * 65535 + 0.5)             # round-to-nearest

FUNCTION dequantize_16(stored, min, max) -> real
  RETURN stored * (max - min) / 65535 + min

FUNCTION quantize_8(value, min, max) -> int (8-bit)
  q := (value - min) / (max - min)
  RETURN floor(q * 255 + 0.5)

FUNCTION dequantize_8(stored, min, max) -> real
  RETURN (stored / 255.0001) * (max - min) + min
```

An **angle** is quantized against the range `[0, 2*pi]` after normalization into that range — 16-bit and 8-bit variants both exist.

A **direction** is a unit vector packed into 16 bits by the shared normal-compression table (see [`_compressed_normal.h`](_compressed_normal.h.md)); reading it yields a unit vector.

A **scaled direction** is a 16-bit packed unit direction followed by a 32-bit float magnitude. Writing one normalizes first; a vector shorter than the epsilon is written as `(0,0,1)` with magnitude zero, so degenerate input round-trips to an explicit zero rather than a NaN.

**Notes** — the dequantizer for 8 bits divides by `255.0001` rather than `255`. The extra ten-thousandth keeps the result strictly inside the range so the range assertion cannot fire on the top code, at the cost of never quite reaching `max`. It is asymmetric with the quantizer and cannot be "fixed" without changing what shipped data decodes to.

## `Writer` flavours

### `MemoryWriter`

**Contract** — accumulates into a growing heap buffer: the capacity starts at 128 bytes and doubles until the write fits. Exposes its buffer and its logical length (which is the high-water mark of the cursor, not the capacity), can be rewound and reused, and can spill itself to a path through the virtual filesystem. Seeking past the end and then writing is legal and extends the logical length.

**Notes** — the distinction between "capacity", "cursor" and "logical length" matters: a writer that seeks backwards to patch a chunk size must not shrink the file.

### `FileWriter`

**Contract** — writes straight to a file. Two modes: shared (any other process may also open it) and exclusive (write-sharing denied). Creates any missing parent directories before opening. Reports whether it opened successfully; a failed open is logged, not fatal, and subsequent writes are silently dropped. Large writes are chunked into 16 MiB pieces so a short write is detected per piece. On close, any read-only attribute the platform put on the file is cleared.

**Notes** — the per-piece write check exists to turn "the disk filled up" into a clear failure rather than a truncated level file.

## Whole-file helpers

These sit *below* the virtual filesystem and take real paths; they exist for bootstrap (reading the filesystem configuration before the filesystem exists) and for the tools.

**`download(path) -> bytes`** — open, read the whole file into one allocation, close. Retries the open once after a short sleep, because a file the engine itself just wrote may still be held briefly by the platform.

**`compress(path, signature, bytes)`** — writes an 8-byte ASCII signature, then the LZ-Huffman stream of the payload. **`decompress(path, signature) -> bytes`** — reads the 8 bytes, refuses (fatally) if they do not match the expected signature, and inflates the rest. The signature is the format's identity check; the length of the compressed body is `file length - 8`.

**`ensure_directories(path)`** — walks the path left to right and creates each directory prefix, ignoring failures for ones that already exist.

## Reader flavours

| Flavour | Backing | Frees on close |
|---|---|---|
| plain | a borrowed range of someone else's buffer | nothing |
| temporary | a heap buffer it was handed | the buffer |
| pack | a memory-mapped region, with the reader's range being a sub-range of it | unmaps the whole region |
| file | the whole file read into one heap buffer | the buffer |
| compressed-file | the whole file inflated into one heap buffer | the buffer |
| mapped, read-only | the file mapped for reading | unmaps, closes the handle |
| mapped, read-write | the file mapped shared for writing | unmaps, closes the handle |

**Contract (mapped readers)** — open the file, ask the platform for its length, map the whole thing, and point the reader at the mapping. A zero-length file is an error: there is nothing to map and every caller expects bytes. The read-write flavour maps *shared*, so stores through the reader's range land in the file — that is how the tools patch assets in place.

**Notes** — the pack flavour keeps the mapping base separately from the reader's own start pointer, because archive entries do not begin on a mapping-granularity boundary: the caller maps from the granularity-aligned offset below the entry and points the reader at the entry itself (see [`LocatorAPI.cpp`](LocatorAPI.cpp.md)).

An optional accounting mode tracks every live mapping with its size and source name, so a leak shows up as a mapping that outlives its level. It is a debugging aid and carries no format weight.
