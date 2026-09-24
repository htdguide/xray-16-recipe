# src/xrGame/rat_state_manager_inline.h

> Replacing the current state in place.

**Needs** — [`rat_state_manager.h`](rat_state_manager.h.md)
**Used by** — [`rat_state_manager.h`](rat_state_manager.h.md)
**Tier floor** — T3: two stack operations

## Purpose

Supplies `change_state`, which is **a pop followed by a push**: the new state takes the
current slot and the state underneath it stays reachable. That is the distinction that
makes the machine a pushdown one — `push_state` deepens the stack, `change_state` does not,
and a state that wants to be returned to after an interruption must be pushed over, not
changed into.

The difference is visible in behaviour. A rat that *pushes* its melee state over its
free-roaming state returns to roaming when the fight ends; a rat that *changes* into its
no-path state gives up the state it was in.
