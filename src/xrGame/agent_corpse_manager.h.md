# src/xrGame/agent_corpse_manager.h

> Declares the squad-level assignment of who reacts to which fallen comrade; behaviour is in [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md).

**Needs** — [`member_corpse.h`](member_corpse.h.md) · [`agent_corpse_manager_inline.h`](agent_corpse_manager_inline.h.md)
**Used by** — [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) · [`agent_corpse_manager_inline.h`](agent_corpse_manager_inline.h.md) · [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares one of the squad brain's four managers: the pending list of squad deaths awaiting
a reactor. Substance is in [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md).

Exported units:

- `register_corpse` — record a squad member's death as pending.
- `react_on_member_death` — match pending deaths to members and hand out the reactions.
- `remove_links` — forget everything naming an object that is leaving the simulation.
- `corpses`, `clear` — the pending list and its reset.
- `update` — an empty per-frame hook, present for uniformity with the other managers.
