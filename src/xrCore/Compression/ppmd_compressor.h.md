# src/xrCore/Compression/ppmd_compressor.h

> Declares the statistical compressor's six entry points: plain, model-trained, and chunked-with-yield.

**Needs** — [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md) · [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md) · [`../fastdelegate.h`](../fastdelegate.h.md)
**Used by** — [`entry_point.cpp`](../../utils/mp_configs_verifyer/entry_point.cpp.md) · [`pch.h`](../../utils/mp_configs_verifyer/pch.h.md) · [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md) · [`Level_network_compressed_updates.cpp`](../../xrGame/Level_network_compressed_updates.cpp.md) · [`configs_dumper.cpp`](../../xrGame/configs_dumper.cpp.md) · [`game_cl_mp.cpp`](../../xrGame/game_cl_mp.cpp.md) · [`xrServer_updates_compressor.cpp`](../../xrGame/xrServer_updates_compressor.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md). Three pairs, each compress-and-decompress:

## Exported units

- **`ppmd_compress` / `ppmd_decompress`** — the whole buffer in one call, starting from an empty model. Simple, and the whole call blocks for as long as it takes.
- **`ppmd_trained_compress` / `ppmd_trained_decompress`** — the same, but the model is primed from a serialized context tree supplied by the caller. Priming costs a pass over the tree and buys substantially better ratios on short inputs, for the same reason a dictionary does in [`rt_compressor9.cpp`](rt_compressor9.cpp.md).
- **`ppmd_compress_mt` / `ppmd_decompress_mt`** — the same work split into independent pieces with a caller-supplied callback invoked between them, so the calling thread can yield or report progress. The suffix promises concurrency and delivers cooperation; see the notes in [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md).

## Notes

The trained variants take the model as a *stream over a buffer*, not as a parsed structure, because the tree is deserialized directly into the coder's own memory pool. The caller therefore only ever holds the serialized bytes.
