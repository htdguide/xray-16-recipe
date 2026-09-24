# src/xrCore/_compressed_normal.h

> Declares the sixteen-bit packing of a unit vector, and the lookup table it needs initialized before first use.

**Needs** — [`_compressed_normal.cpp`](_compressed_normal.cpp.md) · [`xr_types.h`](xr_types.h.md)
**Used by** — [`FS.cpp`](FS.cpp.md) · [`FS.h`](FS.h.md) · [`NET_utils.cpp`](NET_utils.cpp.md) · [`_compressed_normal.cpp`](_compressed_normal.cpp.md) · [`_math.cpp`](_math.cpp.md) · [`vector.h`](vector.h.md)
**Tier floor** — T1: a bit layout that appears on the wire and in shipped vertex data.

## Purpose

Declares the surface implemented in [`_compressed_normal.cpp`](_compressed_normal.cpp.md). Three entry points: compress, decompress, and one initialization that must run before either.

## Exported units

- **`pvCompress`** — a unit vector to a sixteen-bit word.
- **`pvDecompress`** — the word back to a vector.
- **`pvInitializeStatics`** — builds the 8192-entry normalization table the decompression reads. **Must be called once at startup**, before any decompression; [`_math.cpp`](_math.cpp.md) does it. A rebuild that computes the normalization directly rather than by table can drop this, at the cost of a square root per decompression.
