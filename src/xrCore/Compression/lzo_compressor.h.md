# src/xrCore/Compression/lzo_compressor.h

> Declares the dictionary-assisted compressor used for network payloads.

**Needs** — [`lzo_compressor.cpp`](lzo_compressor.cpp.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`lzo_compressor.cpp`](lzo_compressor.cpp.md) · [`traffic_optimization.cpp`](../../xrGame/traffic_optimization.cpp.md) · [`traffic_optimization.h`](../../xrGame/traffic_optimization.h.md)
**Tier floor** — T1: the caller supplies the scratch buffer and must size it from a value the library publishes.

## Purpose

Declares the four entry points implemented in [`lzo_compressor.cpp`](lzo_compressor.cpp.md).

## Exported units

- **`lzo_compress_dict`** — maximum-effort compression of a buffer against a shared dictionary, into a caller-supplied output buffer, using a caller-supplied scratch buffer.
- **`lzo_decompress_dict`** — the bounds-checked inverse.
- **`lzo_initialize`** — one-time library setup; must succeed before anything else here is called.
- **`lzo_get_workmem_size`** — how large the scratch buffer must be for the maximum-effort compressor.

## Notes

The scratch buffer's size is a library constant and is exposed rather than baked in, because it differs between compression levels and between library versions. A rebuild should keep that indirection: hard-coding it is the classic way to get a buffer overrun when the library is updated.
