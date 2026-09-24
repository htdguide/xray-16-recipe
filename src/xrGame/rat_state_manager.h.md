# src/xrGame/rat_state_manager.h

> Declares the rat's pushdown state machine: a table of states, a stack of state identifiers, and the transition rhythm.

**Needs** — [`rat_state_base.h`](rat_state_base.h.md) · [`rat_state_manager_inline.h`](rat_state_manager_inline.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md)
**Used by** — [`ai_rat.cpp`](ai/monsters/rats/ai_rat.cpp.md) · [`ai_rat.h`](ai/monsters/rats/ai_rat.h.md) · [`ai_rat_behaviour.cpp`](ai/monsters/rats/ai_rat_behaviour.cpp.md) · [`rat_state_manager.cpp`](rat_state_manager.cpp.md) · [`rat_state_manager_inline.h`](rat_state_manager_inline.h.md) · [`rat_states.cpp`](rat_states.cpp.md) · [`rat_states.h`](rat_states.h.md)
**Tier floor** — T3: a small map and a stack

## Purpose

Declares the surface implemented in
[`rat_state_manager.cpp`](rat_state_manager.cpp.md), where the transition rule and its
ordering live.

Exported units:

- `construct(rat)` — bind the machine to its rat, before any state is added.
- `add_state(id, state)` — register one state instance under an identifier; the machine
  takes ownership and binds the state to the rat.
- `push_state(id)` — make a state current, remembering what was current.
- `pop_state()` — return to whatever was current before the matching push.
- `change_state(id)` — replace the current state without growing the stack.
- `update()` — run one step of the machine.

The state identifier is a plain integer; the rat supplies the enumeration.
