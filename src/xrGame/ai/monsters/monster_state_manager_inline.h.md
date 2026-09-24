# src/xrGame/ai/monsters/monster_state_manager_inline.h

> The bodies of the root state's forwarding methods, and the two guards that decide whether a creature thinks at all this tick.

**Needs** — [`monster_state_manager.h`](monster_state_manager.h.md) · [`alife_simulator.h`](../../alife_simulator.h.md)
**Used by** — [`monster_state_manager.h`](monster_state_manager.h.md)
**Tier floor** — T3: forwarding, plus two existence checks

## Purpose

Almost all of this is forwarding, and merges into
[`monster_state_manager.h`](monster_state_manager.h.md) in a rebuild. One routine is not
forwarding and is the reason the file matters.

## `update` — the per-tick guards

**Contract** — the entry point the engine's scheduler calls. Runs the creature's selector
unless one of two guards stops it. Does nothing else — the selector is the concrete creature's
`execute`.

```text
FUNCTION update()
  IF the creature's identity is NOT in the alife registry THEN RETURN
  IF the creature is not alive                            THEN RETURN
  run the selector
```

**Invariants** — a creature's brain never runs while it is dead, and never runs while its
authoritative record is absent from the off-screen simulation's registry.

**Notes** — the registry guard is the interesting one. A client object can outlive the removal
of its server record for a short window during the transition between the detailed level and
the coarse world simulation, and during that window every reference the brain would follow —
squad, home, enemies — is liable to name a record that is no longer there. The guard is
described in the source as an addition, which places it as a fix for a crash rather than an
original design decision, but it is now load-bearing: a rebuild whose brains run during that
window will follow dangling references.

It reaches across into the off-screen simulation through a single free function declared
locally rather than through an interface, which is incidental. What must survive is that the
brain asks "does my authoritative record still exist" before thinking.

## `force_script_state` / `execute_script_state`

**Contract** — the first selects a state directly, bypassing the selector; the second runs
whatever is selected without re-selecting. Together they let script hold a creature in a state
across ticks.

## `can_eat`

**Contract** — as declared: a corpse must be selected, and the eating state's start-or-continue
test must pass. The corpse test comes first and short-circuits.

## `should_run`

**Contract** — the start-or-continue rule, stated in full in
[`monster_state_manager.h`](monster_state_manager.h.md).

## `reinit`, `forget_entity`, `critical_finalize`, `current_state_type`, `may_start_control`

**Contract** — pure delegation to the state side of the root's identity. They exist as overrides
only because the two halves of that identity declare the same names and one of them must be
named explicitly. In a rebuild where the manager interface and the state interface do not
collide, none of these five needs to exist.
