# src/xrGame/ai/monsters/states/monster_state_attack.h

> Declares the shared attack behaviour: the nine-way selector every creature that fights hands its combat to.

**Needs** — [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`bloodsucker_attack_state.h`](../bloodsucker/bloodsucker_attack_state.h.md) · [`bloodsucker_attack_state_inline.h`](../bloodsucker/bloodsucker_attack_state_inline.h.md) · [`bloodsucker_state_manager.cpp`](../bloodsucker/bloodsucker_state_manager.cpp.md) · [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`burer_state_attack_melee.h`](../burer/burer_state_attack_melee.h.md) · [`burer_state_attack_melee_inline.h`](../burer/burer_state_attack_melee_inline.h.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`fracture_state_manager.cpp`](../fracture/fracture_state_manager.cpp.md) · [`pseudodog_state_manager.cpp`](../pseudodog/pseudodog_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_controlled_attack.h`](monster_state_controlled_attack.h.md) · _and 2 more_
**Tier floor** — T3: a container state with eight branches and three timers

## Purpose

Declares the surface implemented in
[`monster_state_attack_inline.h`](monster_state_attack_inline.h.md). Every creature in the
chapter except the rat registers one of these under its attack identifier, and the few that
customise combat do it by deriving from this type rather than replacing it — the controlled
attack in [`monster_state_controlled_attack.h`](monster_state_controlled_attack.h.md) is the
example, and the bloodsucker and burer do the same.

## State

```text
RECORD AttackState
  next_run_away_allowed_at : int   # a ten-second cooldown after a completed flight
  behinder_check_started   : int   # see below
  behinder_started         : int   # see below
```

**Invariants** — the two "behinder" timers are **written to zero on entry and never written
again**, and the routine that would advance them is *declared here and defined nowhere*. The
behaviour they belong to — manoeuvring to the enemy's rear — exists only in the group-squad
variant of this state, and this declaration is a copy that was never filled in. Their only
effect is that the flee test's first guard is permanently false. A rebuild should delete all
three.

## Exported units

- construction, in two forms — the default, which builds all nine children, and one that
  accepts a caller-supplied approach and melee state so a creature can substitute its own.
- `initialize` — arm the melee checker and clear the timers.
- `execute` — the selector.
- `setup_substates` — parameterise the flight child.
- The seven private branch tests: steal, find-enemy, flee, run-attack, camp, home-point, and
  the unimplemented behinder.
