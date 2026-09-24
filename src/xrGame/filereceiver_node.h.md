# src/xrGame/filereceiver_node.h

> Declares one inbound transfer session, implemented in [`filereceiver_node.cpp`](filereceiver_node.cpp.md).

**Needs** — [`filetransfer_common.h`](filetransfer_common.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md)
**Used by** — [`file_transfer.cpp`](file_transfer.cpp.md) · [`file_transfer.h`](file_transfer.h.md) · [`filereceiver_node.cpp`](filereceiver_node.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the receiving half of one file transfer. Substance is in
[`filereceiver_node.cpp`](filereceiver_node.cpp.md).

The declaration's own load-bearing content is the **two constructors**, which fix the only
two destinations a transfer may have — a named file the node opens and closes itself, or a
memory buffer the caller owns — and the immutable flag that records which, since it decides
ownership at destruction.

Exported units:

- `filereceiver_node` — one session: a destination, the expected size, an opaque
  caller-defined parameter, and the time of the last chunk.
- Construction from a file name, or from a caller's memory writer.
- `receive_packet` — append a chunk; reads the size header from the first one; reports
  completion.
- `is_complete` — write position equals expected size.
- `signal_callback` — report a status with the current and expected sizes.
- `get_downloaded_size` / `get_writer` / `get_user_param` — accessors. The writer is exposed
  so a caller can detect a failed file open, and so the sites can read the destination once
  the transfer finishes.
- `get_last_read_time` / `set_last_read_time` — the timestamp the sites' timeout reads and
  arms.
- `split_received_to_buffers` — free function: split a completed buffer-list payload back
  into its length-prefixed parts, as views over the original bytes.
