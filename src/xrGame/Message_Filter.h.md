# src/xrGame/Message_Filter.h

> Declares the non-consuming message tap implemented in [`Message_Filter.cpp`](Message_Filter.cpp.md).

**Needs** — [`Message_Filter.cpp`](Message_Filter.cpp.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md)
**Used by** — [`DemoPLay_Control.cpp`](DemoPLay_Control.cpp.md) · [`DemoPlay_Control.h`](DemoPlay_Control.h.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`Message_Filter.cpp`](Message_Filter.cpp.md)
**Tier floor** — T3: a declaration and a small key record

## Purpose

Declares `message_filter`, an observer registry keyed by message kind and sub-kind that
inspects the message stream without consuming it. Substance is in
[`Message_Filter.cpp`](Message_Filter.cpp.md).

Exported units:

- `message_filter` — the tap. Holds the observer map, the optional log file and the
  run-length state for collapsing repeated log lines.
- The observer signature — a callable taking the message kind, the sub-kind and the message
  itself. The message is passed by reference and positioned past its header; the caller
  rewinds afterwards.
- `filter` — register an observer for one (kind, sub-kind) pair. Asserts the pair is free.
- `remove_filter` — unregister it. Asserts the pair is present.
- `check_new_data` — examine one message, unpacking a batch if necessary, and restore the read
  cursor.
- `dbg_set_message_log_file` — open a trace file.
- `msg_type_subtype_t` (private) — the key record. Its ordering uses only the kind and
  sub-kind; the addressed entity and the receive time ride along as decoded output.
- `import` / `dbg_print_msg` (private) — the partial wire decoder and the one-line formatter.

**Notes** — the observer table is capped by a constant the implementation never consults; the
container it actually uses is an ordered array with no fixed capacity. The constant is dead.
