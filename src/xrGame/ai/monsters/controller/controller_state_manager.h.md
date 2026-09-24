# src/xrGame/ai/monsters/controller/controller_state_manager.h

> Declares the controller's brain, implemented in
> [`controller_state_manager.cpp`](controller_state_manager.cpp.md).

**Needs** — [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`controller_state_manager.cpp`](controller_state_manager.cpp.md)
**Used by** — [`controller.cpp`](controller.cpp.md) · [`controller_state_manager.cpp`](controller_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the controller's brain as a specialization of the shared creature brain, so the creature
class can own one without seeing how it decides. The declaration is separate from the body only
because C++ separates them.

## `CStateManagerController`

- **construct** — takes the creature it drives; registers the state set
- **reinit** — reset plus put the creature into its idle demeanour
- **execute** — pick and run one global state
- **remove_links** — forward the "this object is being destroyed, forget it" notice to every
  registered state
- **check_control_start_conditions** — the narrow answer about which external abilities may
  interrupt

Contracts are in the implementation twin.
