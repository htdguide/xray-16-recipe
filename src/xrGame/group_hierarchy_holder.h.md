# src/xrGame/group_hierarchy_holder.h

> Declares the group level of the squad hierarchy: the tier that owns a group's shared perception and its coordination manager.

**Needs** — [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md) · [`group_hierarchy_holder_inline.h`](group_hierarchy_holder_inline.h.md) · [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`agent_manager.h`](agent_manager.h.md)
**Used by** — [`Spectator.cpp`](Spectator.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`ai_rat.h`](ai/monsters/rats/ai_rat.h.md) · [`ai_rat_impl.h`](ai/monsters/rats/ai_rat_impl.h.md) · [`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md) · [`group_hierarchy_holder_inline.h`](group_hierarchy_holder_inline.h.md) · [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md) · [`squad_hierarchy_holder.cpp`](squad_hierarchy_holder.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in
[`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md) and
[`group_hierarchy_holder_inline.h`](group_hierarchy_holder_inline.h.md). A group is the
middle tier of the three-level seniority hierarchy (squad above it, individual members
below): the level at which several creatures share what they have seen, heard and been hit
by, and at which their tactical coordination is planned.

Exported units:

- `CGroupHierarchyHolder` — the group; constructed against the squad it belongs to.
- `register_member` / `unregister_member` — the whole lifecycle; see the implementation for
  the ordering, which is load-bearing.
- `members` — the member list.
- `agent_manager` / `squad` — accessors that assert their target exists.
- Optional leader tracking, compiled in only when the hierarchy is configured to have
  leaders; the shipped build does not.
- Five counters (last action, last action time, active, alive and standing member counts) the
  source itself marks as belonging to one creature type and due for removal. They are public
  fields with no invariant maintained here: whoever writes them owns their meaning.
