# src/xrNetServer/NET_Log.h

> Declares the packet trace log and the record it accumulates.

**Needs** — [`NET_Shared.h`](NET_Shared.h.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`NET_Client.cpp`](NET_Client.cpp.md) · [`NET_Log.cpp`](NET_Log.cpp.md) · [`NET_Server.cpp`](NET_Server.cpp.md) · [`NET_Client.cpp`](empty/NET_Client.cpp.md) · [`NET_Server.cpp`](empty/NET_Server.cpp.md)
**Tier floor** — T3: a buffered file writer.

## Purpose

Declares the surface implemented in [`NET_Log.cpp`](NET_Log.cpp.md).

## Exported units

- **`SLogPacket`** — one trace entry: time, size, type identifier, direction, and a name field
  that is declared and never filled.
- **`INetLog`** — the log itself. Opened with a file name and a base time; records messages
  either from a message record or from a raw span.

## Notes

The log has identity — it owns a file handle and a lock — so it is explicitly non-copyable.
That is the incidental expression of a real constraint: two logs writing one file would
interleave partial lines.

The per-entry name field is 64 bytes, declared, and never written. It costs 64 bytes per
buffered entry and a hundred entries are buffered. A rebuild drops it.
