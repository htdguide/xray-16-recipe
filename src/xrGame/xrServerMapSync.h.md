# src/xrGame/xrServerMapSync.h

> The three answers a server can give when a client says which level it has.

**Needs** — [`xrServerMapSync.cpp`](xrServerMapSync.cpp.md)
**Used by** — [`Level_network_map_sync.cpp`](Level_network_map_sync.cpp.md) · [`xrServerMapSync.cpp`](xrServerMapSync.cpp.md)
**Tier floor** — T1: the values are a frozen one-byte wire tag

## Purpose

One enumeration, and it is a wire format: the value is written as the first byte of the
server's reply, so the numbering is frozen.

## State

```text
ENUM MapSyncResponse : int (8-bit, on the wire)
  success          = 0    # the client has the same level, same version, same geometry
  invalid_checksum = 1    # right level and version, but its geometry does not match
  other_map        = 2    # wrong level or wrong version
```

**Notes** — Three outcomes rather than a boolean, because the *remedy* differs. Having the
wrong level means the client can download the right one, and the server tells it where; having
a corrupted copy of the right level means re-downloading the same thing; success means proceed.
Collapsing them to "no" would leave the client unable to fix itself.

The distinction between wrong-version and corrupt-geometry is also the anti-cheat one: a client
that has edited its level geometry to see through walls reports the right name and version and
the wrong checksum, which lands in the second case.
