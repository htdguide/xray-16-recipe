# src/xrGame/action_planner_inline.h

> Runs a creature's decision cycle: re-solve the plan every update, switch actions only when the plan's first step changes, and persist the whole brain across a save.

**Needs** — [`action_planner.h`](action_planner.h.md) · [`action_base.h`](action_base.h.md) · [`property_storage.h`](property_storage.h.md) · [`property_evaluator.h`](property_evaluator.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`action_planner.h`](action_planner.h.md)
**Tier floor** — T2: a graph search per creature per decision, on the AI's frame budget

## Purpose

This is where a creature decides what to do. The design commitment worth stating plainly,
because it drives everything else: **the plan is recomputed from scratch every update, and
only its first step is ever executed.** Nothing is cached across updates except which
action is currently running. A creature therefore reacts to a changed world on the next
update with no invalidation machinery, at the price of a graph search per creature per
decision — which is exactly why the scheduler's rate degradation exists.

## State

```text
RECORD Planner
  object            : acting object           # the creature this brain drives
  storage           : world-state storage     # owned; shared by reference with every action and evaluator
  initialized       : bool                    # an action is currently running (between initialize and finalize)
  current_action_id : action id               # which one; meaningless unless initialized
  solving           : bool                    # guard: the solver is running right now
  loaded            : bool                    # this brain's state came from a save rather than from scratch
```

Invariants:

- `initialized` false means *no action is between its initialize and its finalize*. The
  pair must balance across the object's whole life, so every path that changes the running
  action finalizes the old one first.
- The structure of the brain — which actions and which evaluators exist, and what their
  preconditions and effects are — may not change while the solver is running. Every
  mutator asserts this. The reason is that the solver holds references into those
  collections while it searches; mutating them mid-search is a use-after-free, and the
  tempting place to do it is inside an evaluator, which the solver calls.
- The world-state storage is cleared on setup, so a re-used brain does not inherit stale
  beliefs.

## `update`

**Contract** — one decision cycle. Solves for a plan; if the plan's first step differs
from what is running, finalizes the old action and initializes the new one; then executes
the running action exactly once. Blocks for the duration of the search. Does not allocate
beyond the solver's own scratch.

```text
FUNCTION update()
  solving = true
  solve()                            # re-plan from the current world state to the goal
  solving = false

  IF solution is empty THEN
    RETURN                           # no action sequence reaches the goal; do nothing this update

  IF an action is already running THEN
    IF running action != solution.first THEN
      running_action.finalize()
      running_action = solution.first
      running_action.initialize()
  ELSE
    running_action = solution.first
    running_action.initialize()

  running_action.execute()
```

**Invariants**

- The action switch compares *identity*, not the whole plan. A plan whose tail changed but
  whose head did not leaves the running action untouched, with no finalize/initialize
  churn. This is the only reason continuous behaviour is possible at all under
  re-planning every update.
- Exactly one execute happens per update, always, including on the update where the
  action changed — which is what guarantees the action base's "initialized actions are
  always executed at least once" rule.

**Notes**

- An empty solution means the goal is unreachable from the current world state: an
  evaluator answers a precondition no action can establish. In the original this is
  treated as a hard error — the brain has no defined behaviour — and a release build
  optionally prints which action was running. The fork added a bare return so that the
  creature simply does nothing that update rather than terminating the process. A rebuild
  should treat an empty plan as a diagnosable but survivable condition, and should log the
  unreachable property, because that is what actually tells an author what they got wrong.
- The solver reports whether the solution *changed* since the last call, which the trace
  uses to print a plan only when it is news. A rebuild wanting readable AI traces needs
  the same flag.

## `finalize`

**Contract** — finalizes the running action and marks the brain uninitialized. Called when
the creature stops thinking — death, going offline, a brain swap — so that the running
action gets its closing call. Must not be called when nothing is running.

## `setup`

**Contract** — binds the acting object, resets the solver, clears the world-state storage,
and marks the brain uninitialized and not loaded. Note that it does *not* finalize a
running action: setup is for a fresh brain, not for re-binding a live one.

## `add_operator` / `add_evaluator`

**Contract** — install an action or an evaluator under an identifier, and immediately bind
the acting object and the shared world-state storage into it. Refuse (by assertion) to run
while the solver is running.

**Invariants** — the binding happens here rather than at the part's construction, which is
what allows actions and evaluators to be constructed before the brain knows which creature
it belongs to. It also means an action that is never added to a planner is never set up,
and will fail its own setup assertions if used.

## `remove_operator` / `remove_evaluator`

**Contract** — withdraw a part by identifier. Refuse while solving. Removing the action
that is currently running is not guarded against and leaves the running-action identifier
dangling — callers avoid it by finalizing first.

## `add_condition` / `add_effect`

**Contract** — declare that an action requires, or establishes, a named world property at
a given truth value. Routed through the planner rather than called on the action directly
so that the "not while solving" guard is applied; the action's own methods have no such
guard.

## `save` / `load`

**Contract** — persists the brain. Writes every evaluator's own state, then every action's
own state, then the world-state storage as a count followed by condition/value pairs.
Reading is the exact mirror and additionally marks the brain as loaded. Both iterate the
parts in their stored order.

**Invariants** — **the set of evaluators and actions must be identical at save and at
load, in the same order.** Nothing in the stream identifies which part a run of bytes
belongs to: the format is positional. Adding an action to a creature therefore invalidates
every existing save of that creature, which is why the save format is version-tagged and
mismatched versions are refused rather than guessed at (see the conformance section).

```text
FUNCTION save(writer)
  FOR EACH evaluator IN evaluators   -> evaluator.save(writer)     # order is the format
  FOR EACH action    IN actions      -> action.save(writer)
  writer.write_int(storage.size)
  FOR EACH (condition, value) IN storage
    writer.write_raw(condition); writer.write_raw(value)

FUNCTION load(reader)
  FOR EACH evaluator IN evaluators   -> evaluator.load(reader)
  FOR EACH action    IN actions      -> action.load(reader)
  count = reader.read_int()
  REPEAT count TIMES
    condition = reader.read_raw(); value = reader.read_raw()
    storage.set(condition, value)
  loaded = true
```

**Notes** — the condition identifier and the value are written as raw memory images rather
than as defined-width fields, so the save format depends on the host's integer widths and
endianness. That is the general policy stated in the system requirements, but it is worth
flagging here because the AI's saved state is otherwise entirely portable data.

## `current_action` / `current_action_id` / `initialized` / `action` / `evaluator` / `object`

**Contract** — plain reads. Asking for the current action when nothing is running is a
contract violation, not a query that returns nothing: callers are expected to know whether
their brain is running.

## Tracing

**Contract** — in debug builds the planner can print, on demand: the current world state
and the goal state as a per-property plus/minus/unknown listing against the evaluator
names; the full brain as a list of evaluators and of operators with each operator's
preconditions and effects; and the plan, printed only when it changes, together with how
many search vertices the solver visited to find it.

**Notes** — the property listing prints a question mark for a property no evaluator has
answered yet, which distinguishes *unknown* from *false*. That distinction exists in the
trace but not in the world state itself, where a property is simply absent until
evaluated; reproducing it is what makes an AI trace readable.
