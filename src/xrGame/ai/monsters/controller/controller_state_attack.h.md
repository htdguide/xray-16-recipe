# src/xrGame/ai/monsters/controller/controller_state_attack.h

> Declares the controller's attack state — the composite that owns every sub-state of a fight — implemented in [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md)
**Used by** — [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md) · [`controller_state_manager.cpp`](controller_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CStateControllerAttack`, the composite state a controller is in for the whole of a
fight. Like every state in chapter 24 it is parameterized over the creature type it drives,
so one state body serves the creature and any variant of it; that parameterization is a C++
convenience and a rebuild may make it a plain interface.

The substance — which sub-states exist, and the selector that chooses among them each tick —
is in the inline file this header includes.

## State

Declared by the generic state base: a current and previous sub-state identifier, the time the
state was entered, the creature it drives, and the sub-state table. See
[`../state.h`](../state.h.md).

Exported units:

- `initialize`, `execute`, `finalize`, `critical_finalize` — the state contract. `execute`
  runs the selector and delegates; `critical_finalize` is the abort path taken when the state
  is left without completing, and every state in the chapter must leave the creature in a
  usable configuration on it.
- `setup_substates` — builds the sub-state table. This is where the fight's repertoire is
  declared; the sub-states in this slice are the camp
  ([`controller_state_attack_camp.h`](controller_state_attack_camp.h.md)), the fast move
  ([`controller_state_attack_fast_move.h`](controller_state_attack_fast_move.h.md)) and the
  psi-bolt fire ([`controller_state_attack_fire.h`](controller_state_attack_fire.h.md)),
  alongside the hide, move-out and control-hit states written elsewhere in this chapter.
- `check_force_state` — the per-tick interrupt: a condition that pre-empts the current
  sub-state regardless of whether it has completed.
- `check_home_point` — whether the creature should return to its authored home position
  rather than continue the fight. Smart-terrain-placed creatures are given a home point and
  will not pursue indefinitely away from it.
- `remove_links` — empty; this state holds no object references across ticks.
