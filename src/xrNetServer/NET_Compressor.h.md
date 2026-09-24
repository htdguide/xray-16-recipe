# src/xrNetServer/NET_Compressor.h

> Declares the per-envelope compressor and the size accounting that decides whether
> compressing anything was worth it.

**Needs** — [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Threads](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`NET_Common.cpp`](NET_Common.cpp.md) · [`NET_Compressor.cpp`](NET_Compressor.cpp.md) · [`NET_Shared.h`](NET_Shared.h.md)
**Tier floor** — T1: callers hand it destination buffers they have already sized, so it works
against raw spans with no ownership.

## Purpose

Declares the surface implemented in [`NET_Compressor.cpp`](NET_Compressor.cpp.md). It exists
as its own type, rather than two free functions, only because it carries a lock and a
statistics table.

## Exported units

- **`worst_case_size(n)`** — the largest output the compressor can produce for `n` bytes of
  input, so the caller can size its destination before compressing. Must be an upper bound,
  never an estimate.
- **`Compress(dest, dest_capacity, src, n)`** — compresses if it helps, copies verbatim if it
  does not, and tags the result either way. Returns the number of bytes written.
- **`Decompress(dest, dest_capacity, src, n)`** — reverses it, verifying the checksum first.
  Returns the number of bytes written.
- **`DumpStats(brief)`** — reports what the compressor achieved, bucketed by input size.

## Notes

The statistics table is keyed by *exact input size*, which is only informative because the
engine's envelopes cluster at a few characteristic sizes. It is off unless a console variable
turns it on, and it costs a map lookup per envelope when on. Diagnostic; a rebuild may drop it
entirely.

Roughly two hundred lines of this file's implementation are a commented-out arithmetic range
coder — the state machine of a statistical compressor that was evaluated and not adopted. It
carries no live decision and a rebuild should not carry it.
