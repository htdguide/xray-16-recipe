# src/xrCore/file_stream_reader.h

> Declares the file-owning variant of the streaming reader, implemented in [`file_stream_reader.cpp`](file_stream_reader.cpp.md).

**Needs** — [`file_stream_reader.cpp`](file_stream_reader.cpp.md) · [`stream_reader.h`](stream_reader.h.md)
**Used by** — [`LocatorAPI.cpp`](LocatorAPI.cpp.md) · [`file_stream_reader.cpp`](file_stream_reader.cpp.md)
**Tier floor** — T1: it holds a platform file handle.

## Purpose

Declares the reader described in [`file_stream_reader.cpp`](file_stream_reader.cpp.md): the sliding-window reader plus ownership of the file it reads.

## Exported units

- **Construct** — open a path and map it whole, with a given window size.
- **Destroy** — unmap, then close the mapping and the file.

Everything else is inherited from [`stream_reader.h`](stream_reader.h.md).
