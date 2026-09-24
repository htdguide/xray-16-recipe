# src/xrGame/ai/monsters/group_states/group_state_panic_run.h

> Declares the pack fleeing rung, implemented in
> [`group_state_panic_run_inline.h`](group_state_panic_run_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_panic_run_inline.h`](group_state_panic_run_inline.h.md)
**Used by** — [`group_state_panic_inline.h`](group_state_panic_inline.h.md) · [`group_state_panic_run_inline.h`](group_state_panic_run_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf state in which a frightened pack creature sprints for the far side of its own
territory. Split from its body only because C++ splits templates that way.

Two constants are authored in the declaration: a creature stops fleeing only once it has been
unseen for 15 seconds **and** is at least 15 world units from its enemy.

## `CStateGroupPanicRun`

- **enter** — prime the path builder
- **execute** — sprint to a spot in the outer home ring away from the enemy
- **is_finished** — the two-part safety test
- **remove_links** — forward the destruction notice

Contracts are in the implementation twin.
