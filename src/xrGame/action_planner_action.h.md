# src/xrGame/action_planner_action.h

> Declares the composite that is simultaneously a planner and an action — the mechanism that makes a creature's behaviour a hierarchy of plans rather than one flat list. Behaviour is in [`action_planner_action_inline.h`](action_planner_action_inline.h.md).

**Needs** — [`action_base.h`](action_base.h.md) · [`action_planner.h`](action_planner.h.md) · [`action_planner_action_inline.h`](action_planner_action_inline.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`action_planner_action_inline.h`](action_planner_action_inline.h.md) · [`action_planner_action_script.cpp`](action_planner_action_script.cpp.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md) · [`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md) · [`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md) · [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares a type that is both a brain and one of the actions a brain can choose. That is
the whole idea: an outer planner picks "fight" as its next step, and "fight" is itself a
planner over "close in", "aim", "fire", "reload". Behaviour composes without any explicit
hierarchy machinery, because the composite satisfies both contracts at once.

Substance is in
[`action_planner_action_inline.h`](action_planner_action_inline.h.md).

Exported units:

- `setup`, `initialize`, `execute`, `finalize`, `completed`, `weight` — the action
  contract, each resolving the ambiguity between the two inherited versions.
- `add_condition` / `add_effect` — forwarded to the planner half, so a sub-action's
  contract is declared through the sub-planner.
- `save` / `load` — planner state first, then action state; the order is the save format.

**Notes** — inheriting from two bases that both define the same lifecycle names is what
forces every method here to exist: each one exists only to say which inherited version
wins, or to call both in a chosen order. A rebuild expressing this as *composition* — an
action that holds a planner — writes the same decisions with none of the disambiguation.
