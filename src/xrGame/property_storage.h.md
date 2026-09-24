# src/xrGame/property_storage.h

> The planner's answer board: the current boolean answer to every question a creature's plan is written against.

**Needs** — [`xrAICore/Navigation/graph_engine_space.h`](../xrAICore/Navigation/graph_engine_space.h.md) · [`property_storage_inline.h`](property_storage_inline.h.md)
**Used by** — [`action_base.h`](action_base.h.md) · [`action_base_inline.h`](action_base_inline.h.md) · [`action_planner.h`](action_planner.h.md) · [`action_planner_inline.h`](action_planner_inline.h.md) · [`property_evaluator.h`](property_evaluator.h.md) · [`property_evaluator_member.h`](property_evaluator_member.h.md) · [`property_storage_inline.h`](property_storage_inline.h.md) · [`property_storage_script.cpp`](property_storage_script.cpp.md) · [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md)
**Tier floor** — T3: a small association list

## Purpose

Evaluators answer questions; operators are written against those answers; the planner
searches between answer sets. All three need one place to read the *current* answers from,
and that is this. It is deliberately a tiny, flat structure — a creature has on the order
of a few dozen live properties, and the planner touches them thousands of times per search.

It is its own file because both the evaluator side
([`property_evaluator.h`](property_evaluator.h.md)) and the operator side depend on it, and
because it is exported to scripts directly.

## State

```text
RECORD PropertyStorage
  entries : list<(question: int (32-bit), answer: bool)>
```

**Invariants**

- At most one entry per question. Writing an existing question replaces its answer; it
  never appends a second.
- The list is **unordered and searched linearly**. This is a decision, not laziness: the
  set is small and hot, entries are appended in the order the planner first asks about
  them, and a linear scan over a contiguous few dozen entries beats a tree. A rebuild that
  substitutes a hash map will be correct and probably slower.
- A question that has never been written does **not** read as false — reading it fails.
  The distinction matters: an unwritten question means the planner was configured without
  the evaluator that answers it, which is an authoring error that must surface, whereas a
  false answer is a legitimate world state.

## `set_property`

**Contract** — records an answer. Replaces in place when the question is already present,
appends otherwise. Never fails.

## `property`

**Contract** — returns the current answer to a question. **Fails** when the question is
unknown; see the invariant above.

## `clear`

**Contract** — forgets every answer. Used when a planner is re-pointed at a different
subject, so that stale answers from the previous one cannot leak into the new plan.

## Script surface

Exported as `property_storage` in
[`property_storage_script.cpp`](property_storage_script.cpp.md): default construction,
`set_property` and `property`. Scripts hold a storage when they write their own evaluators
and operators.
