# src/xrGame/ai/monsters/pseudodog/psy_dog_state_psy_attack_hide.h

> Declares the psi dog's one hiding move: sprint to a cover point chosen relative to the enemy, then stop.

**Needs** — [`psy_dog_state_psy_attack_hide_inline.h`](psy_dog_state_psy_attack_hide_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`psy_dog_state_psy_attack_hide_inline.h`](psy_dog_state_psy_attack_hide_inline.h.md) · [`psy_dog_state_psy_attack_inline.h`](psy_dog_state_psy_attack_inline.h.md)
**Tier floor** — T3: a leaf state holding one destination

## Purpose

Declares the surface implemented in
[`psy_dog_state_psy_attack_hide_inline.h`](psy_dog_state_psy_attack_hide_inline.h.md). It is a
leaf state — it registers no children and therefore fully overrides the tick rather than
delegating — and the only thing it owns is the destination it picked on entry.

## State

```text
RECORD target
  position : vector     # world position of the chosen cover point
  vertex   : int         # its navigation-mesh vertex; the completion test compares against this
```

The pair is stored rather than recomputed because the destination must stay fixed for the
whole run; re-picking each tick would make the dog oscillate between two covers.

## Exported units

- construction — takes the creature; registers no children.
- `initialize` — pick the destination, then prepare the path builder.
- `execute` — one tick of running toward it.
- `check_start_conditions` — always true; the state is selected unconditionally.
- `check_completion` — arrived and stopped.
- `remove_links` — forwards to the container cascade; holds no entity references of its own.
- `select_target_point` (private) — the cover choice, and the only real algorithm here.
