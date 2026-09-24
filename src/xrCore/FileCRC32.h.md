# src/xrCore/FileCRC32.h

> Declares the include-following file checksum.

**Needs** — [`FileCRC32.cpp`](FileCRC32.cpp.md) · [`FS.h`](FS.h.md)
**Used by** — [`r4_shaders.cpp`](../Layers/xrRenderPC_R4/r4_shaders.cpp.md) · [`xrCompressDifference.cpp`](../utils/xrCompress/xrCompressDifference.cpp.md) · [`FileCRC32.cpp`](FileCRC32.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the two entry points implemented in [`FileCRC32.cpp`](FileCRC32.cpp.md).

## Exported units

- **`getFileCrc32`** — checksum this file and, transitively, everything it includes, folding into the caller's running value.
- **`addFileCrc32`** — the same computed independently and *added* to the caller's value, so the result is independent of the order files are visited.
