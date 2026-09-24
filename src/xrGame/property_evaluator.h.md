# src/xrGame/property_evaluator.h

> The planner's question-asking half: an object that answers one boolean question about the world, so the planner can reason about preconditions and effects.

**Needs** — [`property_storage.h`](property_storage.h.md) · [`action_management_config.h`](action_management_config.h.md) · [`property_evaluator_inline.h`](property_evaluator_inline.h.md) · [`xrAICore/Navigation/graph_engine_space.h`](../xrAICore/Navigation/graph_engine_space.h.md)
**Used by** — [`action_planner.h`](action_planner.h.md) · [`action_planner_inline.h`](action_planner_inline.h.md) · [`property_evaluator_const.h`](property_evaluator_const.h.md) · [`property_evaluator_inline.h`](property_evaluator_inline.h.md) · [`property_evaluator_member.h`](property_evaluator_member.h.md) · [`property_evaluator_script.cpp`](property_evaluator_script.cpp.md) · [`script_property_evaluator_wrapper.h`](script_property_evaluator_wrapper.h.md) · [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md)
**Tier floor** — T3: an interface with a two-field state

## Purpose

An **evaluator** is one half of the goal/plan machinery described in the glossary. It
answers exactly one question — *am I hurt?*, *is my weapon loaded?*, *can I see my enemy?*
— as a boolean, against the world as it currently is. **Operators** (actions) are written
with preconditions and effects expressed over those answers, and planning is a search from
the current answers to a goal set of answers.

This file declares what an evaluator *is*, which makes it an interface contract a rebuild
must satisfy: every concrete evaluator in the game, and every evaluator written in Lua,
implements it. The default implementations live in
[`property_evaluator_inline.h`](property_evaluator_inline.h.md); two ready-made
specializations live in [`property_evaluator_const.h`](property_evaluator_const.h.md) and
[`property_evaluator_member.h`](property_evaluator_member.h.md).

## State

```text
RECORD Evaluator
  subject : object          # whose world this evaluator asks about; may be unset until setup
  storage : PropertyStorage # the shared answer board; unset until setup, required by `property`
  name    : text            # identification for the planner's decision log
```

**Invariants**

- `evaluate` may be called only after `setup` has supplied a storage, because an evaluator
  that reads *other* properties (the common case for a composite question) resolves them
  through that storage.
- The answer type is a plain boolean and the question is identified by a small integer.
  Both widths are load-bearing: the planner packs world states as lists of
  (question-identifier, boolean) pairs and compares them by equality, so a rebuild that
  widens the answer to an enumeration must rewrite the state comparison too.
- The evaluator is *stateless with respect to time*: it is re-asked whenever the planner
  needs the answer and must not cache across frames unless it owns the invalidation.

## `evaluate`

**Contract** — returns this evaluator's answer for the current world. No arguments:
everything it needs is the subject and the storage. Must not mutate the world; the planner
calls it many times during one search and assumes the answers are stable within a search.

The base implementation returns *false*, so an evaluator that forgets to override reads as
"the condition does not hold" rather than as an error.

## `setup`

**Contract** — binds the evaluator to a subject and to the storage the planner is
searching over. Called once when the planner installs the evaluator, and again whenever the
planner is re-pointed at a different subject.

## `init`

**Contract** — the construction-time half of `setup`: records the subject and the name, and
leaves the storage unset. Split from `setup` because evaluators are often constructed long
before the planner that will own them exists.

## `property`

**Contract** — reads another question's current answer out of the shared storage. This is
how a composite evaluator is written without reaching into the world twice. Fails if the
question has never been answered — an unanswered question is a planner-configuration error,
not a legitimate "false".

## `Load`

**Contract** — an optional hook for evaluators tuned from a configuration section. The base
does nothing.

## `save` · `load`

**Contract** — an optional hook for evaluators that carry state across a save. The base
writes and reads nothing, which is the correct behaviour for the overwhelming majority:
an evaluator re-derives its answer from the world after a load.

## `CScriptPropertyEvaluator`

**Contract** — the specialization whose subject is a **game object** (the script-visible
facade). This is the type Lua-authored evaluators derive from, and the one exported in
[`property_evaluator_script.cpp`](property_evaluator_script.cpp.md).

**Notes** — the evaluator's name exists only to make the planner's decision log readable.
The original guards it behind a debug switch whose condition has been forced permanently
on, because mod authors debug plans in shipping builds. A rebuild should simply always
carry the name; it costs one reference per evaluator.
