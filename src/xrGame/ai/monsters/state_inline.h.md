# src/xrGame/ai/monsters/state_inline.h

> The container half of a state node: the tick loop that runs one child and reselects when it finishes, the entry/exit sequencing that guarantees a displaced child is torn down, and the cascades that reach the whole tree at once.

**Needs** — [`state.h`](state.h.md) · [`state_defs.h`](state_defs.h.md)
**Used by** — [`state.h`](state.h.md)
**Tier floor** — T3: a tree walk and a small state machine over child identifiers

## Purpose

Every creature brain, every composite behaviour and every leaf in chapter 24 inherits the
five routines on this page. They are short, and they decide more about how monsters behave
than any individual creature file does. Read this before any state file.

Nothing here knows about creatures; the node only knows it owns children and that the
creature it steers is alive.

## `execute` — the tick

**Contract** — runs one tick of this subtree. Re-entrant only in the sense that it is called
every scheduler tick on the same node; it assumes the creature is alive and does not check.

```text
FUNCTION execute()
  check_force_state()               # a subclass may clear current_child here to preempt

  IF current_child IS none
    reselect_state()                # the selector; must set current_child or the tick fails

  child = children[current_child]
  child.execute()

  previous_child = current_child    # recorded BEFORE the completion test, see below

  IF child.check_completion()
    child.finalize()
    current_child = none            # next tick reselects, with previous_child still naming this one
```

**Invariants**

- **Reselection is lazy, not per-tick.** A child, once chosen, keeps running until it reports
  completion or a selector above it forces a change. The default `check_completion` is false,
  so *a node that does not define completion runs forever* and can only be displaced from
  above. Several states in this chapter rely on exactly that.
- **`previous_child` outlives the child.** It is set every tick and never cleared on
  completion, so when the next tick's `reselect_state` runs, `previous_child` names the node
  that just finished. That is why nearly every selector in the chapter reads
  `previous_child == X` to mean "I was doing X" and pairs it with `check_completion` to mean
  "and X is not done yet". A rebuild that clears it on completion breaks every one of them.
- **A leaf must override this.** With no children, `reselect_state` does nothing, the child
  lookup has no key, and the tick fails. Leafness is a contract, not a type.

**Notes** — the debug build, on finding `current_child` still unset after `reselect_state`,
dumps the whole creature's state tree before failing. That exists because a selector with an
unhandled branch — and there are several in the chapter, usually a switch over a classification
with no default arm — leaves the choice unassigned, and the dump is the only way to find out
which node did it.

## `select_state` — entering a child

**Contract** — makes a child current. Idempotent: selecting the child that is already current
does nothing at all, which is what lets selectors re-assert their choice every tick without
restarting the behaviour. Called by `reselect_state` implementations and by brains that
select directly.

```text
FUNCTION select_state(new_child)
  IF current_child == new_child
    RETURN                          # re-asserting a choice is free; this is load-bearing

  IF current_child IS NOT none
    children[current_child].critical_finalize()   # displaced, not completed

  current_child = new_child
  setup_substates()                 # parent parameterises the child...
  children[new_child].initialize()  # ...before the child stamps its clock and clears its own choice
```

**Invariants** — the ordering of the last three steps is the contract: the outgoing child is
torn down *before* the incoming one is parameterised, and parameterisation happens *before*
entry, so a child's `initialize` may already read the parameters it was handed. A rebuild that
initialises first and parameterises second breaks every state whose entry depends on its
target.

**Notes** — the displaced child gets `critical_finalize`, never `finalize`. That distinction is
the only signal a child has that it was interrupted rather than finished, and states that
reserve something (a locked cover point, an overridden animation, a forced enemy) release it
on this path.

## `initialize`, `finalize`, `reset`

**Contract** — `initialize` stamps the entry time and clears both child choices, so a node
re-entered later starts its selector from scratch rather than resuming where it left off.
`finalize` resets. `reset` clears both choices and zeroes the entry time.

**Notes** — the consequence is that **a state tree has no memory across entries.** A creature
that leaves "attack" and comes back re-runs the attack selector from its first branch. Any
persistence a creature appears to have is stored on the creature, not in the tree.

## `critical_finalize` and `reinit` — the cascades

**Contract** — `critical_finalize` tears down the active branch: the current child is
critically finalised (recursively, so the whole live branch unwinds from the leaf up) and this
node resets. `reinit` critically finalises the active branch, then re-initialises *every*
registered child, not just the live ones, then resets.

**Invariants** — `reinit` runs when a creature is respawned or a save is loaded. It must reach
every node, because a node that reserved something during a previous life still holds it. The
active branch is unwound first so that tear-down happens before re-initialisation, never after.

## `remove_links`

**Contract** — forwards an entity-destruction notice to every registered child, live or not.
No return value, no ordering guarantee between siblings.

**Notes** — this is the whole-tree sweep that keeps cached entity references from outliving
their entities. It is pure C++ tax: a rebuild whose references either keep entities alive or
observably expire deletes this routine and its declaration on every state in the chapter,
which is several hundred lines of forwarding.

## `get_state_type`

```text
FUNCTION get_state_type() -> int
  IF children IS empty OR current_child IS none
    RETURN unknown
  deeper = children[current_child].get_state_type()
  RETURN deeper IF deeper IS NOT unknown ELSE current_child
```

**Contract** — names the deepest running node. A leaf reports "unknown", so the answer is the
identifier of the lowest node that still has a running child — which is the leaf's own
identifier, reported by its parent.

## `check_control_start_conditions`

**Contract** — asks the active branch whether a motion-control component may seize the
creature. The container forwards to the current child and permits when the child permits or
when there is no child. Dormant siblings are never consulted, so a veto only ever comes from
the behaviour actually running.

## `fill_data_with`

**Contract** — copies a caller-supplied parameter block into the slot this node was
constructed pointing at. Size is supplied by the caller and not checked against the slot.

**Notes** — see [`state.h`](state.h.md) for what this mechanism is for and what replaces it.
