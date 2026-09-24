# src/xrGame/ai/monsters/states/monster_state_attack_run_attack.h

> Declares the charge: run through the enemy with the attack marker set on the running animation, so the hit lands as the creature passes.

**Needs** — [`monster_state_attack_run_attack_inline.h`](monster_state_attack_run_attack_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`group_state_attack_inline.h`](../group_states/group_state_attack_inline.h.md) · [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) · [`monster_state_attack_run_attack_inline.h`](monster_state_attack_run_attack_inline.h.md)
**Tier floor** — T3: a leaf state that sets one animation parameter

## Purpose

Declares the surface implemented in
[`monster_state_attack_run_attack_inline.h`](monster_state_attack_run_attack_inline.h.md).
Stateless — the only thing it tracks is a success timestamp, and that lives on the creature so
that the animation layer can set it when the hit lands.

## Exported units

- construction — takes the creature; registers no children.
- `initialize` — clear the creature's last-successful-hit stamp.
- `execute` — one tick of the charge.
- `finalize` / `critical_finalize` — pure forwards.
- `check_start_conditions` — the distance-and-angle gate.
- `check_completion` — hit landed, or no longer moving.
- `remove_links` — forwards to the base cascade.
