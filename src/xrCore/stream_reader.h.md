# src/xrCore/stream_reader.h

> Declares the sliding memory-mapped window reader implemented in [`stream_reader.cpp`](stream_reader.cpp.md).

**Needs** — [`stream_reader.cpp`](stream_reader.cpp.md) · [`stream_reader_inline.h`](stream_reader_inline.h.md) · [`FS.h`](FS.h.md) · [`../Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`LocatorAPI.cpp`](LocatorAPI.cpp.md) · [`file_stream_reader.cpp`](file_stream_reader.cpp.md) · [`file_stream_reader.h`](file_stream_reader.h.md) · [`stream_reader.cpp`](stream_reader.cpp.md) · [`stream_reader_inline.h`](stream_reader_inline.h.md) · [`xrTheora_Stream.cpp`](../xrEngine/xrTheora_Stream.cpp.md) · [`xrTheora_Stream.h`](../xrEngine/xrTheora_Stream.h.md) · [`DemoInfo.cpp`](../xrGame/DemoInfo.cpp.md) · [`DemoInfo_Loader.cpp`](../xrGame/DemoInfo_Loader.cpp.md) · [`Level_network_map_sync.cpp`](../xrGame/Level_network_map_sync.cpp.md)
**Tier floor** — T1: it declares a mapping handle and raw cursors into a mapped region.

## Purpose

Declares the streaming reader described in [`stream_reader.cpp`](stream_reader.cpp.md). It shares the typed-read vocabulary (integers of each width, reals, vectors, quantized angles, chunk search) with the whole-file reader by deriving from the same read base — the base supplies everything in terms of "give me N bytes", and this class supplies that one operation plus positioning.

## Exported units

- **Construct** — bind to a file mapping, a region within it, and a window size; maps the first window.
- **Destroy** — unmap. Does not close the mapping; it does not own it.
- **Close** — destroy and release the reader itself.
- **Read** — copy N bytes, crossing windows as needed.
- **Advance / seek / tell / length / elapsed / at end** — positioning; seeking is expressed as a relative advance.
- **Find chunk** — locate a chunk by identifier in the chunked container format and report its payload size and whether it is compression-marked.
- **Open chunk** — a bounded child reader over one chunk's payload; refuses compressed chunks.
- **Read string** — a null-terminated string, interned, copied only when it straddles a window boundary.
- **Mapping handle accessor** — so a child reader can be opened on the same mapping.

**Notes** — The class is explicitly non-copyable: two cursors over one mapped window would each unmap it.
