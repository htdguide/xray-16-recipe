# src/xrCore/FS_internal.h

> Declares the concrete reader and writer flavours, visible only inside the filesystem module.

**Needs** — [`FS.h`](FS.h.md) · [`FS.cpp`](FS.cpp.md) · [`lzhuf.h`](lzhuf.h.md)
**Used by** — [`xr_ini_ex.cpp`](../utils/mp_balancer/xr_ini_ex.cpp.md) · [`FS.cpp`](FS.cpp.md) · [`LocatorAPI.cpp`](LocatorAPI.cpp.md)
**Tier floor** — T1: three of the five flavours own a platform file handle or a mapping and must release it deterministically.

## Purpose

The public surface ([`FS.h`](FS.h.md)) shows callers only "a reader" and "a writer"; which flavour they got — a private buffer, a borrowed range, a mapping, an inflated blob — is the filesystem's business. This file is where the flavours are declared so [`LocatorAPI.cpp`](LocatorAPI.cpp.md) can construct them and nobody else can. The behaviour of each is contracted in [`FS.cpp`](FS.cpp.md) under *Reader flavours* and *Writer flavours*.

## Exported units

- **`CFileWriter`** — writes to a real path, optionally with write-sharing denied; creates missing directories, logs a failed open instead of aborting, and clears the read-only attribute on close.
- **`CTempReader`** — reads a heap buffer it was handed and frees it when closed. This is what a compressed chunk becomes after inflation.
- **`CPackReader`** — reads a sub-range of a memory mapping, remembering the mapping base separately so it can unmap the whole region on close.
- **`CFileReader`** — reads a file slurped whole into memory.
- **`CCompressedReader`** — reads a signature-prefixed LZ-Huffman file inflated whole into memory.
- **`CVirtualFileReader`** — reads a file mapped read-only.
- **`download` / `compress` / `decompress`** — the whole-file helpers, declared here and contracted in [`FS.cpp`](FS.cpp.md).

## Notes

The decision worth carrying over is the *threshold*: a file below a size cut-off is read whole into a heap buffer, and one above it is memory-mapped. The cut-off is 16 KiB for ordinary opens. Mapping a tiny file costs a page-table operation and a fault to save a copy that would have been free; slurping a large one costs a copy and a peak of twice the resident size. See [`LocatorAPI.cpp`](LocatorAPI.cpp.md) for where the choice is made.
