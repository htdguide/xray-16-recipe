# src/xrNetServer/ip_filter.h

> Declares the subnet allow-list consulted before a peer is admitted.

**Needs** — _(none)_
**Used by** — [`NET_Server.cpp`](NET_Server.cpp.md) · [`NET_Server.h`](NET_Server.h.md) · [`NET_Server.cpp`](empty/NET_Server.cpp.md) · [`NET_Server.h`](empty/NET_Server.h.md) · [`ip_filter.cpp`](ip_filter.cpp.md)
**Tier floor** — T2: declares a record of two fixed-width integers.

## Purpose

Declares the surface implemented in [`ip_filter.cpp`](ip_filter.cpp.md).

## Exported units

- **`subnet_item`** — one address block: a base address and a prefix mask, both 32-bit. The
  address is also addressable as four octets, which is only used by the byte-order swap on the
  query path.
- **`ip_filter`** — the list. Three operations: load from configuration, test an address,
  unload.

## Notes

The octets-or-whole-value view of an address appears three times in this module — here, in the
server's address record, and in the parse path — each time written out again. A rebuild
defines an address type once.
