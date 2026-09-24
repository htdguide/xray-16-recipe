# src/xrGame/alife_communication_manager.h

> Declares the off-screen trading layer of the simulation; everything but the constructor is disabled.

**Needs** — [`alife_communication_manager.cpp`](alife_communication_manager.cpp.md) · [`alife_simulator_base.h`](alife_simulator_base.h.md) · [`alife_communication_manager_inline.h`](alife_communication_manager_inline.h.md)
**Used by** — [`alife_communication_manager.cpp`](alife_communication_manager.cpp.md) · [`alife_communication_manager_inline.h`](alife_communication_manager_inline.h.md) · [`alife_interaction_manager.cpp`](alife_interaction_manager.cpp.md) · [`alife_interaction_manager.h`](alife_interaction_manager.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in
[`alife_communication_manager.cpp`](alife_communication_manager.cpp.md). One layer of the
off-screen simulation's inheritance chain, virtual over the shared base like the others.

Exported units:

- **construction** from the server and the simulation's configuration section.

Everything else — the trading buffers, the subset-sum machinery, the capacity checks and
the single public entry point `communicate_with_customer` — is commented out. The cpp twin
describes what it would have done and why a rebuild should not copy its search.

**Notes** — Two constants survive in the disabled declaration and are worth recording: the
subset search's explicit stack is 128 frames deep, and the enumeration of distinct subset
totals stops at 30. Both are caps on an exponential search rather than properties of the
problem.

The build switch named here selects between a slow ownership path that goes through the
graph registry and a fast one that rewrites child lists directly; the fast one is on.
