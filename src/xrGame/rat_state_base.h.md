# src/xrGame/rat_state_base.h

> The interface every rat behaviour state implements: enter, run, leave — over a rat that is handed in once.

**Needs** — [`rat_state_base_inline.h`](rat_state_base_inline.h.md)
**Used by** — [`rat_state_base.cpp`](rat_state_base.cpp.md) · [`rat_state_base_inline.h`](rat_state_base_inline.h.md) · [`rat_state_manager.cpp`](rat_state_manager.cpp.md) · [`rat_state_manager.h`](rat_state_manager.h.md) · [`rat_states.h`](rat_states.h.md)
**Tier floor** — T3: an interface with one back-reference

## Purpose

Rats do not use the goal/plan machinery the humans and the larger monsters use. They run a
plain pushdown state machine, and this is the contract one of its states must satisfy.
Being an interface, the contract itself is what a rebuild must reproduce.

The split from the state machine that drives it
([`rat_state_manager.h`](rat_state_manager.h.md)) is worth keeping: the manager knows
nothing about rats beyond passing the creature through.

## State

```text
RECORD RatState
  subject : Rat     # set once by `construct`, never rebound
```

**Invariants**

- `construct` is called exactly once, by the manager, before any other call. Every other
  method reaches the rat through it and asserts it is set.
- A state is a **singleton per rat**, not per activation: the manager builds one instance
  of each state and reuses it, so a state must not keep per-activation data in itself
  unless it resets that data in `initialize`.
- States are non-copyable, which is the structural expression of the above.

## `initialize`

**Contract** — called once when this state becomes the running state, before its first
`execute`. Where a state stakes out its intent: pick a destination, note the time, choose an
animation.

## `execute`

**Contract** — called every update while this state is the running state, *including* the
update on which it was entered (the manager calls `initialize` and then `execute` in the
same update). This is where the state both acts and decides whether to hand over: a state
changes the machine by pushing, popping or replacing on the manager, then returning.

## `finalize`

**Contract** — called once when a *different* state becomes the running state. Where a
state undoes what it committed to — stops firing, restores a movement mode. Not called when
the machine simply keeps running the same state.

**Notes** — the trio is the same enter/run/leave shape the planner's operators use, and the
same shape the animation actions use. Reproducing the three-call rhythm matters more than
reproducing any individual state, because the whole of
[`rat_states.cpp`](rat_states.cpp.md) is written against it.
