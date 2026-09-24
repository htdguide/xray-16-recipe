# src/xrGame/ai/monsters/chimera/chimera_attack_state.h

> Declares the chimera's attack: the only creature attack in the chapter built entirely out of pounces.

**Needs** — [`state.h`](../state.h.md) · [`chimera_attack_state_inline.h`](chimera_attack_state_inline.h.md) · [`chimera.h`](chimera.h.md)
**Used by** — [`chimera_attack_state_inline.h`](chimera_attack_state_inline.h.md) · [`chimera_state_manager.cpp`](chimera_state_manager.cpp.md)
**Tier floor** — T2: behaviour with geometric target selection against the navigation mesh

## Purpose

Declares the surface implemented in [`chimera_attack_state_inline.h`](chimera_attack_state_inline.h.md). It is a leaf state with no substates but a large internal machine, so its declaration is mostly that machine's fields.

## `ChimeraAttackState`

A leaf state over the shared state contract. It overrides `initialize`, `execute`, `finalize`, `critical_finalize` and the control-arbitration hook, and holds:

```text
RECORD ChimeraAttackState
  phase                  : ENUM { free, rotating, winding_up }
  phase_end_tick         : int      # when winding_up ends
  jump_owner             : handle to the shared jump ability, the thing that
                                    # takes movement away from the path follower
  circle_side            : ENUM { undecided, left, right }
  circle_side_until      : int      # ms; the side is re-drawn when this expires
  attack_jumps_done      : int
  prepare_jumps_done     : int
  last_jump_tick         : int
  stealth_until          : int      # ms; 0 means not stealthing
  target                 : position     # where the creature is heading or pouncing
  target_vertex          : navigation vertex for `target`
  jump_target            : position     # the pounce target chosen before rotating
  is_attack_jump         : bool         # this pounce is meant to land on the enemy
  allow_jump             : bool         # the arbitration latch, see the implementation
  min_run_distance       : real         # derived once at entry
```

Three fields — a predicted target and a separate prepare target and vertex — are declared and never touched. They are the remains of an earlier design that aimed at where the enemy *would* be.
