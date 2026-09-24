# src/xrCore/lzhuf.h

> Declares the LZ-plus-Huffman codec used by the oldest archive generation and by a few legacy tools.

**Needs** — [`LzHuf.cpp`](LzHuf.cpp.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`FS.cpp`](FS.cpp.md) · [`FS_internal.h`](FS_internal.h.md) · [`LzHuf.cpp`](LzHuf.cpp.md)
**Tier floor** — T1: it is a frozen bitstream format read from shipped files.

## Purpose

Declares the codec implemented in [`LzHuf.cpp`](LzHuf.cpp.md): a sliding-window LZ77 stage feeding an adaptive Huffman stage, the compression scheme of the oldest archive generation the engine supports. It predates the compression seam's LZO and DEFLATE paths and exists because those archives still ship.

## Exported units

- **Compress a buffer** — allocates the destination and reports its size.
- **Decompress a buffer** — allocates the destination and reports its size; takes an optional known total size so a stream whose length is recorded elsewhere can be decompressed in one pass. Reports whether it succeeded.
- **Write compressed to a file handle**, and **read compressed from a file handle** — the same codec with the buffer supplied or produced against an already-open file.

## Notes

The format parameters that make the bitstream what it is — the window size, the match-length range and the alphabet size — are fixed in the implementation and are **frozen**: they are the reason a shipped archive decodes. See [`LzHuf.cpp`](LzHuf.cpp.md) for them.

A rebuild that only needs to read the three retail games must still implement this, because the earliest archive generation uses it.
