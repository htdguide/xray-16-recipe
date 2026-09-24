# src/xrGame/agent_enemy_manager.h

> Declares the squad's target assignment; behaviour is in [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md).

**Needs** — [`member_enemy.h`](member_enemy.h.md) · [`agent_enemy_manager_inline.h`](agent_enemy_manager_inline.h.md)
**Used by** — [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_enemy_manager_inline.h`](agent_enemy_manager_inline.h.md) · [`agent_location_manager.cpp`](agent_location_manager.cpp.md) · [`agent_manager.cpp`](agent_manager.cpp.md) · [`agent_manager_actions.cpp`](agent_manager_actions.cpp.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`smart_cover_planner_target_provider.cpp`](smart_cover_planner_target_provider.cpp.md) · [`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md) · [`stalker_kill_wounded_actions.cpp`](stalker_kill_wounded_actions.cpp.md) · [`stalker_property_evaluators.cpp`](stalker_property_evaluators.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the largest of the squad brain's four managers: the pooled enemy list rebuilt
each distribution, the persistent list of who is finishing which wounded enemy, and the two
flags that select between the ordinary and the wounded-only branch. Substance is in
[`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md).

Exported units:

- `distribute_enemies` — the whole decision: pool, rank, assign, permutate, share.
- `useful_enemy` — may this member engage this enemy, given the squad's plan.
- `assigned_wounded` — is this member the one assigned to finish this wounded enemy.
- `wounded_processor` (read and record), `wounded_processed` (read and set) — the
  persistent wounded bookkeeping.
- `enemies` — the pooled list.
- `remove_links` — forget a departing object on either side of a wounded assignment.
- `update` — an empty per-frame hook.

Internal steps given their own names because each is a distinct decision:
`fill_enemies` (pool), `compute_enemy_danger` (rank), `assign_enemies` (greedy),
`permutate_enemies` (distance swaps), `assign_wounded` (the wounded branch),
`assign_enemy_masks` (share back), `evaluate` (the pairwise combat estimate) and
`exchange_enemies` (the swap).

**Notes** — the wounded assignment is stored as a pair of pairs: the enemy, then its
assigned member's identifier and a done flag. The member is held by *identifier* while the
enemy is held by reference, because the member may go offline and be looked up again while
the enemy is by construction still in the simulation. A rebuild should hold both the same
way and look both up.
