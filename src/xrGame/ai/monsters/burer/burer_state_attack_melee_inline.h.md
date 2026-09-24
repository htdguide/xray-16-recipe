# src/xrGame/ai/monsters/burer/burer_state_attack_melee_inline.h

> The shared close-quarters attack, fenced by two distances — and unreachable, because nothing selects it.

**Needs** — [`burer_state_attack_melee.h`](burer_state_attack_melee.h.md) · [`monster_state_attack.h`](../states/monster_state_attack.h.md)
**Used by** — [`burer_state_attack_melee.h`](burer_state_attack_melee.h.md)
**Tier floor** — T3: two predicates

## Purpose

An entire behaviour expressed as two numbers wrapped around a shared state: the burer would close and strike inside five units, and stop once the enemy got beyond nine. The gap between the two is hysteresis — enter at five, leave at nine — so the creature does not flicker in and out at a single boundary.

**Nothing selects it.** The attack tree registers it in its substate table and its arbitration never names it; see [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md). A rebuild may drop it, but the entry/exit-hysteresis idea it encodes is used elsewhere in the chapter and worth keeping in mind.

## State

Stateless.

```text
enter_distance = 5 world units
leave_distance = 9 world units
```

## `check_start_conditions`

**Contract** — True only while the enemy is within five units.

## `check_completion`

**Contract** — True once the enemy is beyond nine units.

**Notes** — Both read the enemy's current position with no null guard, so the state assumes an enemy exists — which is safe because the burer's attack tree is only ever entered with one.
