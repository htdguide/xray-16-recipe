# src/xrCore/Compression/lzo_compressor.cpp

> Thin pass-through to the compression library's dictionary-assisted entry points.

**Needs** — [`lzo_compressor.h`](lzo_compressor.h.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`lzo_compressor.h`](lzo_compressor.h.md)
**Tier floor** — T1.

## Purpose

Exists only so the rest of the engine never names the compression library's types. Four functions, each a single delegation. The decision worth recording is *which* of the library's several entry points were chosen.

## `lzo_compress_dict`

**Contract** — compress a buffer at the library's **maximum effort level** against a shared dictionary, writing into a caller-supplied output buffer whose size is passed in and updated to the actual length on return. Requires a scratch buffer of the size the library publishes. Slow — this is the level used when compressing once and decompressing many times.

## `lzo_decompress_dict`

**Contract** — the inverse, using the library's **bounds-checked** decompressor rather than the fast one. The output size is passed in as the buffer's capacity and updated to the real length. A malformed input is detected and reported rather than overrunning.

**Notes** — the safe variant is chosen deliberately: this path decompresses network payloads, which come from an untrusted peer. Every other decompression in the engine reads data the engine itself wrote or shipped, and uses the fast variant.

## `lzo_initialize` / `lzo_get_workmem_size`

**Contract** — one-time library setup, and the scratch-buffer size the maximum-effort compressor needs.

## Notes

A **dictionary** here is a fixed block of bytes both ends already have, which the compressor may reference as if it preceded the input. For many small, highly similar messages — which is exactly what a game's network traffic is — it is the difference between compression helping and hurting, because a short message has no history of its own to match against. The dictionary itself is a data file; see [`rt_compressor9.cpp`](rt_compressor9.cpp.md) for where it is loaded from.
