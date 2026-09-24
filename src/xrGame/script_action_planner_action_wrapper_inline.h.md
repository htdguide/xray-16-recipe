# src/xrGame/script_action_planner_action_wrapper_inline.h

> The nested-planner adapter's constructor.

**Needs** — [`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md)
**Used by** — [`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md)
**Tier floor** — T2

## Purpose

One constructor, forwarding a subject and a name to the composite operator it specializes.
Both default to nothing and to empty text, so a script may declare a sub-planner before it
knows which creature it belongs to and bind the subject later through `setup`.

**Notes** — the name labels the whole sub-plan in the planner's trace, so naming these well
is what makes a hierarchical AI trace readable at all: the outer plan prints the composite's
name, and the inner plan prints beneath it.
