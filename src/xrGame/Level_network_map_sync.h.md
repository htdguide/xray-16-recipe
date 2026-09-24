# src/xrGame/Level_network_map_sync.h

> Declares the record holding one connection attempt's map-verification state, implemented in [`Level_network_map_sync.cpp`](Level_network_map_sync.cpp.md).

**Needs** — [`Level_network_map_sync.cpp`](Level_network_map_sync.cpp.md)
**Used by** — [`Level.h`](Level.h.md) · [`Level_network_map_sync.cpp`](Level_network_map_sync.cpp.md)
**Tier floor** — T3: a plain record of flags and strings

## Purpose

Declares `LevelMapSyncData`, the state of the "do we both have this level" handshake: what
the client announced, what the server answered, and how long it has been waiting. The level
owns one by value. Substance — including the record's fields and its two invariants — is in
[`Level_network_map_sync.cpp`](Level_network_map_sync.cpp.md).

Exported units:

- `LevelMapSyncData` — the record. Every field is cleared on construction; the constructor
  is the only place the initial state is stated.
- `CheckToSendMapSync` — sends the map announcement once per attempt.
- `ReceiveServerMapSync` — decodes the server's verdict into the two failure flags.
- `IsInvalidMapOrVersion` / `IsInvalidClientChecksum` — read those flags. Two predicates
  rather than one because the caller treats the two failures differently: a wrong map
  reconnects, a wrong checksum stalls.
