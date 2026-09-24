# src/xrGame/ai/ai_monsters_misc.h

> Declares the group tactical vote, and defines the transition vocabulary of the engine's older, stack-based creature state machine.

**Needs** — [`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md)
**Used by** — [`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md) · [`monster_enemy_manager.cpp`](monsters/monster_enemy_manager.cpp.md) · [`rat_state_switch.cpp`](monsters/rats/rat_state_switch.cpp.md) · [`stalker_property_evaluators.cpp`](../stalker_property_evaluators.cpp.md)
**Tier floor** — T3: a state stack and two function declarations

## Purpose

Declares the two routines implemented in
[`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md), and — the part that is substance rather
than declaration — fixes the *transition vocabulary* of the stack-based state machine that
the older creature brains are written in. That vocabulary is the reason a reader can follow
a creature's think routine at all, so it is written down here in full even though in the
original it is a family of textual substitutions.

## State

The vocabulary presumes that the creature owning it holds exactly this much:

```text
RECORD StackBrain
  state_stack   : list<state id>   # used as a stack; never empty while the brain runs
  current_state : state id         # invariant: == top of state_stack
  stop_thinking : bool             # set when this tick's decision is final
```

**Invariants** — `current_state` is always the top of the stack; every transition that
touches one touches the other. The stack is never popped empty — a state that returns to
its predecessor must have been pushed by one. The brain's think loop runs state handlers
repeatedly until `stop_thinking` is set, so a handler that neither transitions nor stops
thinking spins forever.

## The transition vocabulary

Six transitions, in two families of three. Each is written as a statement that *ends the
handler*: control returns to the think loop immediately, which is why they read as verbs
rather than as assignments.

```text
replace(s)     : state_stack.top = s ; current_state = s ; RETURN
                 # same depth, different state — the predecessor is forgotten

return_to_prev : state_stack.pop() ; current_state = state_stack.top ; RETURN
                 # this state is finished; resume whatever pushed it

descend(s)     : state_stack.push(s) ; current_state = s ; RETURN
                 # nested state — the predecessor is remembered and will resume
```

**The two families** — each of the three has a *deferred* and an *immediate* form. The
deferred form leaves `stop_thinking` as the handler found it, so the think loop's next pass
happens on the next tick. The immediate form clears `stop_thinking` first, so the loop runs
the *new* state's handler within the same tick. That distinction is load-bearing: it is how
a creature can traverse several states in one update when the intermediate ones are pure
decisions with no visible duration, without ever giving a state a "zero-length" special
case.

Each also has a conditional wrapper — "if this predicate holds, take this transition" —
which exists only to keep the handlers readable; it decides nothing a plain conditional
would not.

**Notes** — the recorded-decision helper ("write the creature's state to the log") also
sets `stop_thinking`, which means that in the original the *debug trace statement and the
end-of-thinking marker are the same statement*. That is why it appears at the end of state
handlers that otherwise do nothing: in a non-debug build it reduces to refreshing the
creature's view of nearby dynamic objects and stopping. A rebuild must keep the refresh and
the stop, and may drop the trace.

## `choose_action`

**Contract** — see [`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md). The default arguments
declared here are load-bearing: with no asking entity the whole group votes, and the
default group radius is 100 world units.

## `group_can_beat`

**Contract** — see [`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md).

**Notes** — the declaration here and the definition in the implementation file disagree on
the collection type of the enemy list — a set here, a sequence there. It links only because
nothing outside the implementation file calls it. A rebuild has one function with one
signature; the *sequence* is the one that matters, because the algorithm is
order-dependent.
