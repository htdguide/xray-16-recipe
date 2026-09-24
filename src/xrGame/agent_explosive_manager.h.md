# src/xrGame/agent_explosive_manager.h

> Declares the squad's live-explosive tracking and reaction assignment; behaviour is in [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md).

**Needs** — [`danger_explosive.h`](danger_explosive.h.md) · [`agent_explosive_manager_inline.h`](agent_explosive_manager_inline.h.md)
**Used by** — [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`agent_explosive_manager_inline.h`](agent_explosive_manager_inline.h.md) · [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) · [`ai_stalker_misc.cpp`](ai/stalker/ai_stalker_misc.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares one of the squad brain's four managers: the pending list of noticed explosives
and the set of explosives this squad has already dealt with. Substance is in
[`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md).

Exported units:

- `register_explosive` — notice an explosive: suppress duplicates, queue a reaction, and
  register a danger area for the whole squad.
- `react_on_explosives` — match pending explosives to members and hand out the reactions.
- `remove_links` — forget an object leaving the simulation, from both the queue and the
  suppression set.
- `update` — an empty per-frame hook.

**Notes** — the suppression set holds entity identifiers while the pending list holds
object references. The split is what allows the suppression to outlive the queue entry:
an explosive stays suppressed after its reaction has been handed out, and only a departure
from the simulation clears it.
