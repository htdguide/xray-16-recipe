# src/xrGame/ai/monsters/state.h

> The contract every creature state satisfies: one node in a tree that can act as a leaf, as a container of other nodes, or as a whole brain, with no type distinction between the three.

**Needs** — [`state_defs.h`](state_defs.h.md) · [`state_inline.h`](state_inline.h.md) · [`control_com_defs.h`](control_com_defs.h.md)
**Used by** — [`bloodsucker_attack_state.h`](bloodsucker/bloodsucker_attack_state.h.md) · [`bloodsucker_attack_state_hide.h`](bloodsucker/bloodsucker_attack_state_hide.h.md) · [`bloodsucker_attack_state_inline.h`](bloodsucker/bloodsucker_attack_state_inline.h.md) · [`bloodsucker_predator.h`](bloodsucker/bloodsucker_predator.h.md) · [`bloodsucker_predator_lite.h`](bloodsucker/bloodsucker_predator_lite.h.md) · [`bloodsucker_state_capture_jump.h`](bloodsucker/bloodsucker_state_capture_jump.h.md) · [`bloodsucker_vampire.h`](bloodsucker/bloodsucker_vampire.h.md) · [`bloodsucker_vampire_approach.h`](bloodsucker/bloodsucker_vampire_approach.h.md) · [`bloodsucker_vampire_approach_inline.h`](bloodsucker/bloodsucker_vampire_approach_inline.h.md) · [`bloodsucker_vampire_execute.h`](bloodsucker/bloodsucker_vampire_execute.h.md) · [`bloodsucker_vampire_execute_inline.h`](bloodsucker/bloodsucker_vampire_execute_inline.h.md) · [`bloodsucker_vampire_hide.h`](bloodsucker/bloodsucker_vampire_hide.h.md) · [`bloodsucker_vampire_hide_inline.h`](bloodsucker/bloodsucker_vampire_hide_inline.h.md) · [`bloodsucker_vampire_inline.h`](bloodsucker/bloodsucker_vampire_inline.h.md) · _and 115 more_
**Tier floor** — T3: a node type with virtual hooks and a child table; nothing device-facing

## Purpose

Chapter 24's whole vocabulary is this one type. A creature's brain, the "attack" behaviour
inside it, and the "run at the enemy" behaviour inside *that* are all the same kind of node,
differing only in whether they registered children. That uniformity is the load-bearing idea:
a state can be promoted from leaf to container, or reused at a different depth, without
changing its interface, and every creature's brain is assembled by composing nodes rather than
by writing a dispatcher.

The file declares the node; [`state_inline.h`](state_inline.h.md) implements the *container*
half of it — the per-tick loop, the enter/exit sequencing, the cascade operations. This page
describes what an implementor owes; that page describes what it gets for free.

## State

```text
RECORD StateNode
  owner            : reference to the creature this node steers
  children         : map<int, StateNode>    # registered at construction, immutable after
  current_child    : optional<int>          # none means "reselect on the next tick"
  previous_child   : optional<int>          # what ran last tick; the selector's memory
  started_at       : int                    # global clock reading when this node was entered
  shared_slot      : optional<reference>    # see `fill_data_with`
```

**Invariants**

- `current_child` is `none` or a key present in `children`. Looking up a key that was never
  registered is a programming error, not a runtime condition; there is no fallback node.
- A node with no children must override `execute`, because the inherited container `execute`
  would look up child `none` and fail. Leafness is enforced by behaviour, not by the type.
- `previous_child` is written *after* the child runs, so a selector reading it during
  selection sees the previous tick's choice. Every "continue what I was doing" test in the
  chapter is written against that.
- A node owns its children and destroys them; registration transfers ownership.

## The per-tick contract

The nine overridable points, in the order a tick may reach them. A rebuild that keeps only
these names and this ordering has kept the chapter's spine.

```text
FUNCTION check_force_state()          # unconditional preemption: may reselect before anything runs
FUNCTION reselect_state()             # choose a child when none is current; the selector
FUNCTION initialize()                 # entered: stamp the clock, clear the child choice
FUNCTION setup_substates()            # called after a child is chosen, before it is entered:
                                      #   the parent's one chance to hand it parameters
FUNCTION execute()                    # one tick of behaviour
FUNCTION check_completion() -> bool   # "I am done"; default false, so a node runs until displaced
FUNCTION check_start_conditions() -> bool  # "I could run now"; default true
FUNCTION finalize()                   # left because it completed
FUNCTION critical_finalize()          # left because something displaced it
FUNCTION reinit()                     # the creature was respawned or reloaded
FUNCTION remove_links(object)         # an entity is being destroyed; drop every reference to it
FUNCTION check_control_start_conditions(kind) -> bool  # veto on a motion-control request
```

**The two exits are not the same exit, and confusing them is the classic bug.** `finalize`
runs when the node itself reported completion; `critical_finalize` runs when a selector above
it chose somebody else mid-behaviour. Anything a node reserved — a locked cover point, an
overridden animation, a forced enemy — must be released on *both* paths, and several states in
this chapter release on both by overriding both and calling the same teardown.

**`remove_links` is mandatory and unimplementable by default.** Every node must declare it,
even when its body only forwards to the container's cascade. The reason is that states cache
raw references to other entities (an enemy, a corpse, a controller) and an entity can be
destroyed at any point in the frame; the cascade is how a destruction sweeps the whole tree.
A rebuild with a reference model that survives destruction can delete this hook entirely — it
is the chapter's largest single piece of incidental machinery.

## `check_control_start_conditions`

**Contract** — asked before a motion-control component (a jump, a threaten animation, an
anti-aim step) is allowed to seize the creature. Returns whether the *currently active* branch
of the tree permits it. The container implementation forwards to the current child and
defaults to permitting; a node that must not be interrupted answers false. The veto is
consulted top-down along the active branch only, so a dormant sibling cannot block anything.

## `get_state_type`

**Contract** — reports the identifier of the deepest currently-running node, so the debug
overlay and the script-facing queries can name what a creature is doing in one value. Walks
the active branch to its leaf and reports that leaf's identifier, falling back to the node's
own when the branch bottoms out. Reports "unknown" when nothing is running.

## `fill_data_with`

**Contract** — copies a block of parameters into a slot the child node was constructed
pointing at. This is how `setup_substates` parameterises a generic child ("move to a point",
"look at a point", "perform an action") differently at each site that reuses it.

**Notes** — the mechanism is a raw byte copy into a slot whose type the parent and child agree
on by convention and nothing checks. What it *solves* is that the chapter has perhaps a dozen
generic leaf behaviours reused at fifty sites, each needing a different parameter record. A
rebuild expresses this as a typed parameter passed at selection time and loses nothing;
keeping the untyped slot buys only the ability to register the child once and re-point it
every tick.

## `time_started`

**Contract** — the global clock reading at which this node was entered. Read by completion
tests that are really timeouts, and by tests asking "was I hit since I got here".
