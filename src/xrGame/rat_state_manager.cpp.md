# src/xrGame/rat_state_manager.cpp

> Runs the rat's state machine: one step per update, with enter and leave fired only on an actual change.

**Needs** — [`rat_state_manager.h`](rat_state_manager.h.md) · [`rat_state_base.h`](rat_state_base.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — reached through its declarations in [`rat_state_manager.h`](rat_state_manager.h.md); callers name that, not this file.
**Tier floor** — T3: a map lookup and a stack top per update

## Purpose

The transition rule for rat behaviour. A rat's states are registered once at spawn and
selected by pushing and popping identifiers; this file decides, each update, whether the
selection changed and therefore whether anyone's enter and leave hooks fire.

## State

```text
RECORD RatStateManager
  subject       : Rat
  states        : map<int, RatState>     # owned; destroyed with the manager
  stack         : stack<int>             # the selection, innermost on top
  last_state_id : int                    # what actually ran last update; starts unset
```

**Invariants**

- `last_state_id` tracks what *ran*, not what is on top of the stack. They differ for
  exactly one update after a push or a pop, and closing that gap is what `update` does.
- The stack is never empty during `update`. The rat pushes its initial state at spawn.
- An identifier may be registered once; registering a second state under the same
  identifier is rejected, and pushing an unregistered identifier is rejected, both at the
  point of the mistake rather than at the point of use.
- The manager owns its states and destroys them with itself.

## `update`

**Contract** — advances the machine one step. Runs the current state's `execute` exactly
once per call, and fires `finalize` on the outgoing state and `initialize` on the incoming
state when — and only when — the top of the stack differs from what ran last.

```text
FUNCTION update()
  new_id = stack.top                      # requires a non-empty stack

  IF new_id == last_state_id THEN
    states[new_id].execute()
    RETURN

  # A change: leave the old state before entering the new one, and give the new
  # state its first execute in the SAME update, so a state that transitions on
  # entry costs no frame.
  IF a state ran before THEN states[last_state_id].finalize()
  last_state_id = new_id
  states[new_id].initialize()
  states[new_id].execute()
```

**Invariants** — the order `finalize` → `initialize` → `execute` is load-bearing. Leaving
before entering means the outgoing state's cleanup (stop firing, release a movement mode)
is observed by the incoming state's setup. And entering *plus* executing in one update is
what lets a state chain — the rat's attack state can hand straight to its pathing state
without the rat standing still for a frame.

**Notes**

- Because `execute` runs *after* the state may have pushed or popped, a state's own
  transition takes effect on the next update, not inside the current one. A state must
  therefore return immediately after transitioning; every state in
  [`rat_states.cpp`](rat_states.cpp.md) does.
- A pop that uncovers the state that was already running fires no hooks, because the
  comparison is against `last_state_id`. Deliberate: pushing a state that immediately pops
  is a no-op rather than a spurious re-entry.
- The initial `last_state_id` is the all-ones sentinel, which no real state uses, so the
  first update always counts as a change.

## `push_state` · `pop_state` · `add_state` · `construct`

**Contract** — `push_state` requires the identifier to be registered. `pop_state` requires
a non-empty stack; it does not check that anything remains underneath, so popping the last
state leaves the next `update` with nothing to run. `add_state` binds the state to the rat
as it registers it, which is why `construct` must run first.
