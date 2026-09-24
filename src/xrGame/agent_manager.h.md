# src/xrGame/agent_manager.h

> Declares the squad brain and its accessors for the seven subordinate managers.

**Needs** — [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_inline.h`](agent_manager_inline.h.md)
**Used by** — [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) · [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`agent_location_manager.cpp`](agent_location_manager.cpp.md) · [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) · [`agent_manager_inline.h`](agent_manager_inline.h.md) · [`agent_manager_planner.cpp`](agent_manager_planner.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`ai_stalker.cpp`](ai/stalker/ai_stalker.cpp.md) · [`ai_stalker_debug.cpp`](ai/stalker/ai_stalker_debug.cpp.md) · _and 22 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`agent_manager.cpp`](agent_manager.cpp.md): the squad
brain's lifecycle, its single update entry point, and typed access to each of its parts.

Exported units:

- **`AgentManager`** — construct, destroy, `update`, `remove_links`, `cName` (the fixed
  name `agent_manager`, used by the scheduler and by debug output).
- **Accessors** — `corpse`, `enemy`, `explosive`, `location`, `member`, `memory`, `brain`,
  each returning the subordinate by reference. Defined in
  [`agent_manager_inline.h`](agent_manager_inline.h.md).

**Notes** — Whether this type participates in the engine's scheduler is a compile-time
choice declared here; see the cpp twin for what that choice decides.
