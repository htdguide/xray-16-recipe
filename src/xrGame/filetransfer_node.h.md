# src/xrGame/filetransfer_node.h

> Declares one outbound transfer session and the four sources it can read from, implemented in [`filetransfer_node.cpp`](filetransfer_node.cpp.md).

**Needs** — [`filetransfer_common.h`](filetransfer_common.h.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md) · [`xrCommon/xr_deque.h`](../xrCommon/xr_deque.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md)
**Used by** — [`file_transfer.cpp`](file_transfer.cpp.md) · [`file_transfer.h`](file_transfer.h.md) · [`filetransfer_node.cpp`](filetransfer_node.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the sending half of a file transfer. Substance is in
[`filetransfer_node.cpp`](filetransfer_node.cpp.md).

Its own load-bearing content is the **source interface**, which is what makes the transfer
node source-agnostic. Five questions, and every source must answer all five:

```text
INTERFACE FileReader
  make_data_packet(packet, chunk_size) -> at_end   # append up to chunk_size bytes
  is_first_packet() -> bool                        # read position is zero
  size()  -> int                                   # total to be sent
  tell()  -> int                                   # sent so far
  opened() -> bool                                 # the source is usable
```

`is_first_packet` is the interesting member: the size-and-parameter header is written on the
first chunk, and the *source* is the only thing that knows whether this is the first one.
Making it part of the interface is what keeps that decision out of the node.

Exported units:

- `file_reader` — the interface above.
- `disk_file_reader` — a file, opened through the virtual filesystem.
- `memory_reader` — a fixed block of memory owned elsewhere.
- `buffers_vector_reader` — a list of buffers, each sent with a four-byte length prefix and
  reported in a total that includes the prefixes.
- `memory_writer_reader` — a buffer still being appended to, with a promised final size;
  the relay case.
- `filetransfer_node` — one session: a source, the current chunk size, the rate controller's
  history, the opaque caller parameter and the progress callback.
- Four constructors, one per source kind.
- `calculate_chunk_size` — the adaptive rate: additive increase while the peak rises, random
  re-probe once it stops, pinned at the maximum on the authoritative side.
- `make_data_packet` — the next chunk, with the header on the first.
- `is_complete` / `is_ready_to_send` / `opened` / `get_chunk_size` — state queries.
- `signal_callback` — report a status with the sent and total sizes.
