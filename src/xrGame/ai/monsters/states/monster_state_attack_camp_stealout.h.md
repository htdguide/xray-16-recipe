# src/xrGame/ai/monsters/states/monster_state_attack_camp_stealout.h

> Declares the ambush's creeping phase: leave cover at a stalk and move to where the enemy was last seen.

**Needs** — [`monster_state_attack_camp_stealout_inline.h`](monster_state_attack_camp_stealout_inline.h.md) · [`monster_state_move.h`](monster_state_move.h.md)
**Used by** — [`monster_state_attack_camp_inline.h`](monster_state_attack_camp_inline.h.md) · [`monster_state_attack_camp_stealout_inline.h`](monster_state_attack_camp_stealout_inline.h.md)
**Tier floor** — T3: a leaf state with four completion tests

## Purpose

Declares the surface implemented in
[`monster_state_attack_camp_stealout_inline.h`](monster_state_attack_camp_stealout_inline.h.md).
It derives from the shared move-state base rather than from the bare state node, which is where
its timing field comes from; it holds no state of its own.

## Exported units

- construction — takes the creature; registers no children.
- `execute` — one tick of creeping.
- `check_start_conditions` — whether there is anywhere to creep to.
- `check_completion` — the four ways it ends.
- `remove_links` — forwards to the base cascade.
