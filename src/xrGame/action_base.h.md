# src/xrGame/action_base.h

> Declares the planner *operator* every creature action derives from: the fixed lifecycle, the world-state access and the edge cost. Behaviour is in [`action_base_inline.h`](action_base_inline.h.md).

**Needs** — [`action_management_config.h`](action_management_config.h.md) · [`property_storage.h`](property_storage.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`action_base_inline.h`](action_base_inline.h.md) · [`xrAICore/Components/operator_abstract.h`](../xrAICore/Components/operator_abstract.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`action_base_inline.h`](action_base_inline.h.md) · [`action_base_script.cpp`](action_base_script.cpp.md) · [`action_planner.h`](action_planner.h.md) · [`action_planner_action.h`](action_planner_action.h.md) · [`action_planner_action_inline.h`](action_planner_action_inline.h.md) · [`action_planner_inline.h`](action_planner_inline.h.md) · [`action_planner_script.cpp`](action_planner_script.cpp.md) · [`action_script_base.h`](action_script_base.h.md) · [`agent_manager_actions.h`](agent_manager_actions.h.md) · [`object_actions.h`](object_actions.h.md) · [`script_action_wrapper.h`](script_action_wrapper.h.md) · [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md)
**Tier floor** — T3: a declaration; the parameterization over the acting object is a language convenience

## Purpose

Every concrete thing a creature can *do* — approach, aim, fire, take cover, open a door,
say a line — is an operator in the planner's sense: a named action with preconditions over
world-state properties, effects on those properties, a cost, and a body that runs while it
is the chosen action. This declares the half of that which is common to all of them, on
top of the abstract operator the planner core defines.

It is parameterized over the type of object it acts on so that a stalker's actions are
typed against a stalker and a monster's against a monster, with no cast at each use. The
script-facing instantiation — actions written in Lua — is fixed to the game object facade
and is named here.

Substance is in [`action_base_inline.h`](action_base_inline.h.md).

Exported units:

- The lifecycle: `setup`, `initialize`, `execute`, `finalize` — and the state enumeration
  naming its five stages, which exists only for the debug log.
- `weight` and `set_weight` — the planner's edge cost for this operator.
- `set_property` / `property` — read and write the shared world state.
- `set_inertia_time`, `inertia_time`, `start_level_time`, `completed` — the hysteresis
  that stops the planner from abandoning an action the instant it starts.
- `first_time` — whether this is the first update since the action was chosen.
- `save` / `load` — empty by default; an action with state overrides them.
- The acting object and the world-state storage, both publicly readable because script
  actions reach them directly.
- A registration helper type whose sole purpose is to carry the script export; see
  [`action_base_script.cpp`](action_base_script.cpp.md).

**Notes**

- The action's name is carried purely for diagnostics and is empty in most instantiations.
  In a release build nothing reads it.
- The debug-only logging members are compiled in or out by
  [`action_management_config.h`](action_management_config.h.md), which means the class
  *layout* differs between debug and release builds. That is a build hazard in the
  original — a debug game module cannot be mixed with release ones — and a rebuild should
  keep diagnostics out of the object's shape.
