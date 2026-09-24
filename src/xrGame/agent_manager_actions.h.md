# src/xrGame/agent_manager_actions.h

> Declares the four squad-level operators.

**Needs** — [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) · [`action_base.h`](action_base.h.md)
**Used by** — [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) · [`agent_manager_planner.cpp`](agent_manager_planner.cpp.md)
**Tier floor** — T3: declarations.

## Purpose

Declares the surface implemented in
[`agent_manager_actions.cpp`](agent_manager_actions.cpp.md). Each is the engine's generic
planner action with the squad manager as its object, and each names a squad mode.

Exported units:

- **`AgentManagerActionBase`** — the generic action bound to the squad manager; the base
  of the other four.
- **`NoOrders`** — quiet mode; overrides the exit hook.
- **`GatherItems`** — looting mode; no overrides.
- **`KillEnemy`** — combat mode; overrides entry, exit and per-cycle.
- **`ReactOnDanger`** — danger mode; overrides entry and per-cycle.

Each is constructed with the squad manager it acts on and a name used in plan traces.
