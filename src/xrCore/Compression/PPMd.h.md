# src/xrCore/Compression/PPMd.h

> Declares the four entry points of the statistical coder: bring up its pool, tear it down, encode a stream, decode a stream.

**Needs** — [`PPMdType.h`](PPMdType.h.md) · [`Model.cpp`](Model.cpp.md)
**Used by** — [`Model.cpp`](Model.cpp.md) · [`SubAlloc.hpp`](SubAlloc.hpp.md) · [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md)
**Tier floor** — T1: the surface is four calls, but every one of them acts on one process-wide pool of raw memory whose size the caller states in megabytes.

## Purpose

This is the boundary between the engine and an adopted public-domain compressor. Everything behind it — the context tree, the range coder, the pool — is [`Model.cpp`](Model.cpp.md)'s business; everything in front of it is [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md)'s. The header exists so the engine can call the coder without seeing its internals, and it is the natural place to record what the coder demands of a caller.

## Exported units

- **`StartSubAllocator(megabytes)`** — allocate the coder's private pool. Idempotent when the requested size already matches. Reports failure rather than aborting.
- **`StopSubAllocator()`** — release the pool. Safe to call when none is held.
- **`GetUsedMemory()`** — bytes of the pool currently occupied by the model. Informational, and also the number the model's own restoration policy consults when deciding whether a restart is worth it.
- **`EncodeFile(out, in, max_order, restoration)`** — read `in` to exhaustion, write the compressed stream to `out`.
- **`DecodeFile(out, in, max_order, restoration)`** — the mirror. Stops when the encoder's terminating escape appears.
- **`PrintInfo(decoded, encoded)`** — *imported*, not exported: the coder calls it periodically and the engine supplies an empty body. A rebuild should delete it, or make it the progress hook it was meant to be.

## Restoration policy

```text
ENUM RestorationMethod
  restart    # throw the model away and start over  (the engine's choice)
  cut_off    # prune the context tree; the source notes it is nearly twice as slow
  freeze     # stop growing the tree and keep coding against it; the source calls
             # this dangerous
```

**Invariants** — `max_order` and the restoration policy are **part of the stream**, not tuning: both sides derive the same model from the same byte history, and any disagreement about when the model is rebuilt desynchronizes them at that byte and every byte after. The pool size is in the same class, because the pool filling is what triggers restoration.

`max_order == 1` has a special meaning the header spells out: it does *not* restart the model, so a sequence of encodes can share one accumulated model across several inputs — a solid archive. The engine never uses it; the capability is why the parameter is per-call rather than per-pool.

**Notes** — the entry points are named for files and typed for file handles because the original was a command-line archiver. The engine feeds them memory cursors instead (see [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)). A rebuild should name them for streams and pass the stream explicitly rather than reaching a global.
