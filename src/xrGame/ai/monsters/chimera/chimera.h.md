# src/xrGame/ai/monsters/chimera/chimera.h

> Declares the chimera: a base creature whose whole attack is a pounce, tuned by seven authored numbers.

**Needs** — [`base_monster.h`](../basemonster/base_monster.h.md) · [`chimera.cpp`](chimera.cpp.md)
**Used by** — [`chimera.cpp`](chimera.cpp.md) · [`chimera_attack_state.h`](chimera_attack_state.h.md) · [`chimera_attack_state_inline.h`](chimera_attack_state_inline.h.md) · [`chimera_script.cpp`](chimera_script.cpp.md) · [`chimera_state_manager.cpp`](chimera_state_manager.cpp.md)
**Tier floor** — T2: a creature class

## Purpose

Declares the surface implemented in [`chimera.cpp`](chimera.cpp.md), and — unusually for a creature header — declares the record that holds the creature's authored attack tuning, because the attack state reads it directly.

## `Chimera`

Beyond the base lifecycle points it declares:

- `get_attack_params` — hands the attack state the authored tuning record described below. This is the whole interface between the creature and its behaviour; the attack state touches no other chimera-specific member.
- `jump` — launch a scripted pounce at a position with a strength factor, and roar. Exposed so the script layer can stage a pounce.
- `HitEntityInJump` — the damage a landed pounce deals.
- `CustomVelocityIndex2Action` — maps the chimera's two extra velocity profiles back to abstract actions, so the animation layer can pick a clip for a path segment moving at a chimera-only speed.
- `get_monster_class_name` — returns `"chimera"`.

It also holds two velocity profiles of its own, for the fast spin and for the pounce's launch.

## `AttackParams`

The authored record, read once in `Load` from the creature's configuration section, read every tick by the attack state.

```text
RECORD AttackParams
  attack_radius         : real   # the ring around the enemy the creature works on
  prepare_jump_timeout  : int    # ms between repositioning pounces
  attack_jump_timeout   : int    # ms between damaging pounces
  stealth_timeout       : int    # ms of crouched waiting granted when unseen from behind
  force_attack_distance : real   # authored, unread — see the implementation twin
  num_attack_jumps      : int    # damaging pounces before the creature must reposition
  num_prepare_jumps     : int    # repositioning pounces before it may attack again
```
