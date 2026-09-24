# src/xrGame/script_action_wrapper_inline.h

> The script operator adapter's constructor.

**Needs** — [`script_action_wrapper.h`](script_action_wrapper.h.md)
**Used by** — [`script_action_wrapper.h`](script_action_wrapper.h.md)
**Tier floor** — T2

## Purpose

One constructor, forwarding a subject and a name to the operator it specializes. Both default
to nothing and to empty text, so a script may declare an operator before it knows which
creature it belongs to and bind the subject later through `setup` — which is how the planner
installs its parts (see [`action_planner_inline.h`](action_planner_inline.h.md)).

**Notes** — the name is a diagnostic label. It appears in the planner's trace and in the
message emitted when a script's cost function answers below the permitted floor, which is the
only place a default-constructed operator's mistakes become hard to attribute.
