# src/xrGame/ai/monsters/states/monster_state_controlled_attack.h

> Declares the puppet's attack: the ordinary combat behaviour, with the target overridden from outside for as long as it runs.

**Needs** — [`monster_state_controlled_attack_inline.h`](monster_state_controlled_attack_inline.h.md) · [`monster_state_attack.h`](monster_state_attack.h.md)
**Used by** — [`monster_state_controlled_attack_inline.h`](monster_state_controlled_attack_inline.h.md) · [`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md)
**Tier floor** — T3: a derived state that overrides only entry and exit

## Purpose

Declares the surface implemented in
[`monster_state_controlled_attack_inline.h`](monster_state_controlled_attack_inline.h.md). It
derives from the shared attack state rather than replacing it, which is the design worth
noticing: a puppeted creature fights *exactly* as it normally would — same chain, same charge,
same flight rules — and the only difference is which entity its enemy manager reports.

## Exported units

- construction — takes the creature; inherits all nine children of the base attack state.
- `initialize` / `execute` — force the enemy before deferring to the base.
- `finalize` / `critical_finalize` — release the forced enemy.
- `get_enemy` (private) — read the target the controller wrote.
