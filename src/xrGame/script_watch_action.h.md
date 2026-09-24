# src/xrGame/script_watch_action.h

> Declares the look-at order a script attaches to an entity action: what to aim the head (or a searchlight) at.

**Needs** — [`script_abstract_action.h`](script_abstract_action.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`script_entity_action.h`](script_entity_action.h.md) · [`script_watch_action.cpp`](script_watch_action.cpp.md) · [`script_watch_action_inline.h`](script_watch_action_inline.h.md) · [`script_watch_action_script.cpp`](script_watch_action_script.cpp.md) · [`searchlight.cpp`](searchlight.cpp.md)
**Tier floor** — T3: an order record

## Purpose

Declares the surface whose bodies are in
[`script_watch_action_inline.h`](script_watch_action_inline.h.md) and
[`script_watch_action.cpp`](script_watch_action.cpp.md), and whose script registration is
in [`script_watch_action_script.cpp`](script_watch_action_script.cpp.md).

## Exported units

- **The goal-type enumeration** — four ways a look order can be expressed: at an object,
  by sight type alone, in a direction, or "keep looking where you already are".
- **The order record itself** — the fields a consumer reads, all public because two
  unrelated consumers (a creature's sight manager and a searchlight) read different
  subsets directly rather than through accessors.
- **Six constructors** — see the inline twin; each fixes a different combination of goal
  type and payload.
- **Four setters** — watch object, watch type, watch direction, watch bone.
- **`initialize`** — present and empty; the action interface demands it.

## Notes

The record carries two searchlight-only fields (a target point and a per-axis rotation
speed pair) that the creature path never reads, and a bone name the searchlight path never
reads. The type is a union of two orders that were never separated. A rebuild is free to
split them, provided both remain constructible from script under the single exported name
`look`.
