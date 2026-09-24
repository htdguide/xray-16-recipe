# src/xrGame/ai/monsters/control_run_attack.h

> Declares the run-through attack, implemented in [`control_run_attack.cpp`](control_run_attack.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_run_attack.cpp`](control_run_attack.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlRunAttack`. Substance is in
[`control_run_attack.cpp`](control_run_attack.cpp.md).

## State

Four authored numbers — a distance band to the enemy and a cooldown window — and the
timestamp of the next permitted attack. The ability has **no** channel payload: it is the
only custom element in this slice with nothing for its caller to fill in, which is why its
clip is looked up by a literal name instead.

Exported units: `load`, `reinit`, `activate`, `on_release`, `on_event`,
`check_start_conditions`.
