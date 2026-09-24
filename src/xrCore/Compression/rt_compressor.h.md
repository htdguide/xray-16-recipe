# src/xrCore/Compression/rt_compressor.h

> Declares the two real-time compressors: the fast one used everywhere, and the strong one used for network traffic.

**Needs** — [`rt_compressor.cpp`](rt_compressor.cpp.md) · [`rt_compressor9.cpp`](rt_compressor9.cpp.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`xrCompress.cpp`](../../utils/xrCompress/xrCompress.cpp.md) · [`rt_compressor.cpp`](rt_compressor.cpp.md) · [`rt_compressor9.cpp`](rt_compressor9.cpp.md) · [`LocatorAPI.cpp`](../LocatorAPI.cpp.md) · [`xrCore.cpp`](../xrCore.cpp.md) · [`xrCore.h`](../xrCore.h.md)
**Tier floor** — T1: the caller allocates the output buffer and must size it from the worst-case-expansion formula below.

## Purpose

Declares two compressor pairs with the same shape and different trade-offs, implemented in [`rt_compressor.cpp`](rt_compressor.cpp.md) and [`rt_compressor9.cpp`](rt_compressor9.cpp.md).

## Exported units

**The fast pair** — one-shot compress and decompress at the library's speed-optimized level, with a per-thread scratch buffer, no dictionary, and no setup beyond a one-time library initialization. **This is the pair that decompresses archive entries**, so it is on the level-loading critical path and its decompressor is called millions of times per run.

**The strong pair** — one-shot compress and decompress at the maximum-effort level, against an optional shared dictionary loaded from the game data. Used for network payloads, where the same bytes are compressed once and sent to many peers. Has explicit setup and teardown because it owns the dictionary.

**The size bound** — both pairs expose the same worst-case output size for a given input, and it is the same formula:

```text
worst_case(n) = n + n/64 + 16 + 3
```

**Invariants** — the caller **must** allocate at least this much, even though compression almost always produces less. Incompressible input makes the algorithm emit literal runs with a small per-block header, and this is the bound on that overhead. Sizing the output buffer to the input length is the mistake this function exists to prevent.

## Notes

The two pairs use the same underlying format, so anything the strong compressor produces the fast decompressor can read and vice versa — the difference is entirely in how hard the compressor searches. The dictionary is the one exception: a payload compressed against a dictionary requires the same dictionary to decompress.
