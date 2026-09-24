# src/xrCore/Compression/compression_ppmd_stream.h

> A byte cursor over a memory buffer, shaped to look like the file handle the statistical coder expects.

**Needs** — [`compression_ppmd_stream_inline.h`](compression_ppmd_stream_inline.h.md) · [`PPMdType.h`](PPMdType.h.md)
**Used by** — [`Coder.hpp`](Coder.hpp.md) · [`Model.cpp`](Model.cpp.md) · [`PPMdType.h`](PPMdType.h.md) · [`compression_ppmd_stream_inline.h`](compression_ppmd_stream_inline.h.md) · [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md) · [`ppmd_compressor.h`](ppmd_compressor.h.md) · [`xrCore.cpp`](../xrCore.cpp.md) · [`traffic_optimization.cpp`](../../xrGame/traffic_optimization.cpp.md) · [`traffic_optimization.h`](../../xrGame/traffic_optimization.h.md)
**Tier floor** — T2: a pointer and two bounds.

## Purpose

The statistical coder ([`Model.cpp`](Model.cpp.md)) was written against a file interface: get a byte, put a byte, end-of-input. The engine never compresses to a file; it compresses between buffers. This is the adapter, and it is what the coder's byte macros resolve to.

The header declares it; [`compression_ppmd_stream_inline.h`](compression_ppmd_stream_inline.h.md) defines the operations. The split is incidental — one type, two files, because the type has to be declared before the coder's macros reference it and defined after.

## Exported units

- **`stream`** — the cursor: construct over a buffer and its length, put one byte, get one byte, rewind, expose the buffer, report the offset.
