# src/xrGame/actor_mp_state.h

> Declares the networked player state record and the holder that writes and reads it; the wire format is in [`actor_mp_state.cpp`](actor_mp_state.cpp.md).

**Needs** — [`actor_mp_state.cpp`](actor_mp_state.cpp.md) · [`actor_mp_state_inline.h`](actor_mp_state_inline.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`actor_mp_client.h`](actor_mp_client.h.md) · [`actor_mp_client_export.cpp`](actor_mp_client_export.cpp.md) · [`actor_mp_client_import.cpp`](actor_mp_client_import.cpp.md) · [`actor_mp_server.cpp`](actor_mp_server.cpp.md) · [`actor_mp_server.h`](actor_mp_server.h.md) · [`actor_mp_server_export.cpp`](actor_mp_server_export.cpp.md) · [`actor_mp_server_import.cpp`](actor_mp_server_import.cpp.md) · [`actor_mp_state.cpp`](actor_mp_state.cpp.md) · [`actor_mp_state_inline.h`](actor_mp_state_inline.h.md)
**Tier floor** — T1: the record's bit fields are the wire layout's small-field word

## Purpose

Declares the full state of a networked player as one flat record, and the holder that
serializes it. The record is larger than what goes on the wire — it carries the whole
rigid-body state so that the same record can describe a ragdolling player — and the wire
form selects from it.

Substance is in [`actor_mp_state.cpp`](actor_mp_state.cpp.md).

## State

```text
RECORD NetworkedPlayerState
  physics_orientation       : quaternion
  physics_angular_velocity  : vector
  physics_linear_velocity   : vector
  physics_force             : vector
  physics_torque            : vector
  physics_position          : vector
  position                  : vector      # a duplicate of the physics position
  logic_acceleration        : vector      # the intent-driven acceleration, distinct from physics
  model_yaw                 : real
  camera_yaw, camera_pitch, camera_roll : real
  time                      : int
  health                    : real        # normalized to [0,1] on the wire
  radiation                 : real        # normalized to [0,1] on the wire
  inventory_active_slot     : int (4 bits)
  body_state_flags          : int (15 bits)
  physics_state_enabled     : bool (1 bit)
```

Invariants:

- The three small fields are declared at their exact wire widths and packed into one
  machine word. Their widths are the format, not an optimization.
- The plain position and the physics position are the same value; the reader assigns one
  from the other after reading. The duplication is historical and a rebuild should keep
  one.
- The rigid-body force, torque, angular velocity and orientation are in the record but
  **not on the wire**. A ragdolling player's physics is sent through the separate physics
  synchronization path, and these fields are filled from there.

Exported units:

- `write` / `read` — the wire form.
- `relevant` — store a new state and report whether it is worth sending (always yes in the
  shipped configuration).
- `state` — the held state.

**Notes** — several fields are marked in the source as "should be removed": the plain
position, the three camera angles and the timestamp. The camera angles are duplicated by
the body state elsewhere, and the timestamp by the packet's own. They are still sent, so
they are still part of the format.
