# src/xrGame/traffic_optimization.cpp

> Loading the two pre-trained compression models that shrink multiplayer update packets.

**Needs** — [`traffic_optimization.h`](traffic_optimization.h.md) · [`xrCore/Compression/compression_ppmd_stream.h`](../xrCore/Compression/compression_ppmd_stream.h.md) · [`xrCore/Compression/lzo_compressor.h`](../xrCore/Compression/lzo_compressor.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`traffic_optimization.h`](traffic_optimization.h.md)
**Tier floor** — T1: the working buffers are handed to compressors that require a specific
address alignment and raw byte extents.

## Purpose

Network update packets in a multiplayer session are highly repetitive — the same entity
identifiers, the same field layouts, the same small deltas, thousands of times a second.
General-purpose compression on a packet that size achieves almost nothing, because the
packet is too short to contain its own statistics.

The answer is to **ship the statistics**. Two model files are authored offline from recorded
traffic and shipped with the game: a statistical model for one compressor, and a dictionary
for the other. Each is loaded once at session start and reused for every packet, so a
hundred-byte update is compressed against a model built from megabytes of representative
traffic.

This file is only that loading. It has no compression logic of its own — the compressors are
core services — and the choice of which to use is the session's, expressed through the flag
set in [`traffic_optimization.h`](traffic_optimization.h.md).

## State

```text
RECORD LzoDictionary
  data : bytes
  size : int
```

The statistical model is held by its stream object, which takes ownership of the loaded
bytes but not of freeing them; see the teardown note below.

## `init_ppmd_trained_stream`

**Contract** — loads the trained statistical model from a fixed path under the game's
configuration root and wraps it in a stream. Returns whether it succeeded; a missing file is
logged and reported, not fatal — a session simply runs without that compressor. Blocks on
the filesystem. Allocates a buffer the size of the whole file and keeps it for the session.

```text
FUNCTION init_ppmd_trained_stream() -> result
  path := resolve("$game_config$", "mp/ppmd_updates.mdl")
  IF NOT exists(path)
    log and RETURN failure
  buffer := read whole file
  stream := new trained_stream(buffer, length(buffer))
  RETURN success
```

**Invariants** — the model is loaded whole into memory rather than streamed, because it is
consulted for every packet and a filesystem read per packet is not affordable. The path is
fixed and is part of the shipped data layout.

## `deinit_ppmd_trained_stream`

**Contract** — rewinds the stream, recovers the buffer it was built over, frees the buffer,
then destroys the stream.

**Invariants** — the order is the contract. The stream does not own its backing bytes: it was
handed a buffer and it must be asked for that buffer back before it is destroyed, or the
buffer leaks. A rebuild giving the stream ownership of its bytes removes this dance entirely,
which is the better design.

## `init_lzo`

**Contract** — loads the dictionary from a fixed path, initializes the dictionary-based
compressor, and allocates its scratch working memory **aligned to a sixteen-byte boundary**.
Returns whether it succeeded; a missing file is logged and reported, not fatal.

```text
FUNCTION init_lzo() -> (working_memory, raw_allocation, dictionary)
  path := resolve("$game_config$", "mp/lzo_updates.dic")
  IF NOT exists(path)
    log and RETURN failure
  dictionary := read whole file
  compressor.initialize()
  raw_allocation := allocate(compressor.workmem_size + 16)
  working_memory := the first 16-byte-aligned address at or after raw_allocation + 16
  RETURN success
```

**Invariants** — two buffers are returned where one was allocated, and both must be kept:
the aligned one is what the compressor is given, the raw one is what must eventually be
freed. This is the file's one genuinely T1 obligation — the compressor requires its scratch
memory at an aligned address, the allocator does not promise one, and the gap is bridged by
over-allocating and stepping forward.

The over-allocation is a full sixteen bytes rather than fifteen, and the alignment is
computed from the address *after* that step. Fifteen would suffice for the alignment itself;
the extra byte is unexplained and harmless.

## `deinit_lzo`

**Contract** — frees the raw allocation and the dictionary. Must be given the *raw*
allocation, not the aligned pointer.

## Notes

The pair of files this module loads are the only shipped data whose content is derived from
recorded play rather than authored. A rebuild that changes the update packet layout
invalidates both: the models are trained against a specific byte structure, and applying
them to a different one compresses worse than not compressing at all. Either retrain them or
ship without them — the loaders already treat absence as a supported configuration.
