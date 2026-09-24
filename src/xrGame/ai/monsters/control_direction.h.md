# src/xrGame/ai/monsters/control_direction.h

> Declares the direction resource and its payload, implemented in [`control_direction.cpp`](control_direction.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`control_direction.cpp`](control_direction.cpp.md) · [`control_direction_base.cpp`](control_direction_base.cpp.md) · [`control_manager.cpp`](control_manager.cpp.md) · [`control_manager.h`](control_manager.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlDirection`, the pure element on the direction channel, together with the
two records that cross the channel boundary. Substance is in
[`control_direction.cpp`](control_direction.cpp.md).

## State

`SControlDirectionData` — the channel payload a capturer writes: a target angle and a
target speed for each of heading and pitch, plus a flag asking that the turn rate be
scaled by the body's progress toward its target speed. Described in the implementation
twin.

`SRotationEventData` — the payload of the rotation-end event: a bit set naming which of the
two axes just arrived. Both may arrive in the same frame and the event fires once carrying
both.

Exported units:

- `reinit`, `update_frame` — the lifecycle and the per-frame integration.
- `is_face_target` (by position and by object), `is_from_right` (by position and by
  angle), `is_turning`, `get_heading`, `get_heading_current`, `angle_to_target` — the
  query surface the rest of the chapter uses to ask about the creature's facing.
- `pitch_correction` — private; derives the pitch target from the ground or the path.
