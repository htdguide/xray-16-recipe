# src/xrCore/client_id.h

> A connected client's identity: a 32-bit number in a type of its own, so it cannot be confused with an entity identifier or a frame number.

**Needs** — [`xr_types.h`](xr_types.h.md)
**Used by** — [`NET_utils.cpp`](NET_utils.cpp.md) · [`net_utils.h`](net_utils.h.md) · [`game_base_script.cpp`](../xrGame/game_base_script.cpp.md) · [`game_cl_base.h`](../xrGame/game_cl_base.h.md) · [`game_sv_deathmatch.h`](../xrGame/game_sv_deathmatch.h.md) · [`game_sv_event_queue.h`](../xrGame/game_sv_event_queue.h.md) · [`NET_Shared.h`](../xrNetServer/NET_Shared.h.md)
**Tier floor** — T1: it is byte-packed because it is written into network packets and save records as a raw image.

## Purpose

The multiplayer layer passes client identities through dozens of call sites alongside entity identifiers, sector numbers and frame counters, all of which are also 32-bit integers. Wrapping it makes those confusions compile errors. That is the file's entire reason to exist, and a rebuild with a distinct-type facility does it in one line.

## State

```text
RECORD ClientId
  value : int (32-bit)     # 0 is the unset/invalid identity
```

Byte-packed with no padding: the value is written into packets by raw copy.

## Exported units

- **Construct** — default to zero, or from a number.
- **Value / set** — read and write the number.
- **Compare** — equality, inequality, and ordering, all on the number, so it can key an ordered container.

**Notes** — Zero means "no client" by convention throughout the multiplayer code; nothing here enforces it.
