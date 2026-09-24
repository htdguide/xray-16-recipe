# src/xrGame/doors_door.h

> Declares one door's state machine and geometry, implemented in [`doors_door.cpp`](doors_door.cpp.md).

**Needs** — [`doors.h`](doors.h.md) · [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md)
**Used by** — [`doors_actor.cpp`](doors_actor.cpp.md) · [`doors_door.cpp`](doors_door.cpp.md) · [`doors_manager.cpp`](doors_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `doors::door`, the record the manager keeps for one physics object that behaves as
a door. Substance in [`doors_door.cpp`](doors_door.cpp.md).

Exported units:

- construction from a physics object, and destruction — which notifies every creature
  currently holding the door.
- `change_state(initiator, state)` — the one verb, meaning both *claim this state* and
  *release my claim*, depending on what the door is already trying to do.
- `on_change_state(state)` — the world reporting that the door arrived somewhere.
- `lock` / `unlock` / `is_locked(state)` — the script lock.
- `is_blocked(state)` — whether somebody else has claimed the opposite state.
- `position` / `get_matrix` / `get_vector(state)` — the hinge's registered position, the
  object's live transform, and the leaf's tip in each state.
- in debug builds: the authored name, the list of current claimants, and a membership test —
  the only way to see why a door is not moving.

**Notes** — the three state fields (current, target, previous) and the claimant list are
private and the class is non-copyable. The record is owned by the manager and referred to by
handle everywhere else.
