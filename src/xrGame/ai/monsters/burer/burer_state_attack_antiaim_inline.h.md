# src/xrGame/ai/monsters/burer/burer_state_attack_antiaim_inline.h

> Twelve lines of behaviour: start the anti-aim ability, stand and face the enemy while it runs, finish when it finishes.

**Needs** — [`burer_state_attack_antiaim.h`](burer_state_attack_antiaim.h.md) · [`anti_aim_ability.h`](../anti_aim_ability.h.md) · [`burer.h`](burer.h.md)
**Used by** — [`burer_state_attack_antiaim.h`](burer_state_attack_antiaim.h.md)
**Tier floor** — T3: a wrapper around an ability

## Purpose

The thin seam between the burer's attack tree and the shared anti-aim ability — the thing that punishes a player for lining up a shot on a burer by draining their stamina and knocking the weapon out of their hands (the drain itself is [`burer.cpp`](burer.cpp.md)'s `StaminaHit`).

The state exists because the ability cannot decide for itself when to fire: the tree must own that. So the state's whole job is to hold the arbitration latch open for exactly as long as it takes to activate the ability, then get out of the way.

## State

```text
allow_anti_aim : bool   # raised only around the activation call
```

## `initialize`

**Contract** — Raises the latch, activates the anti-aim ability through the creature's control manager, lowers the latch. Asserts the ability is now running — if it is not, the tree polled a condition that then stopped holding, and continuing would leave the state waiting forever.

```text
FUNCTION initialize()
  base.initialize()
  allow_anti_aim = true
  control.activate(anti_aim)
  allow_anti_aim = false
  ASSERT anti_aim.is_active()
```

**Notes** — The latch is raised and lowered around a single call because the control manager asks the *current state* for permission synchronously during activation. Raising it for longer would let the ability restart itself on a later tick without the tree's consent.

## `execute`

**Contract** — Face the enemy (turning only if not already roughly aimed) and stand. The ability owns everything else, including the animation.

## `check_start_conditions`

**Contract** — Delegates entirely to the ability's own readiness test, which is what detects that the player is aiming.

## `check_completion`

**Contract** — Finished as soon as the ability is no longer active. The state has no timeout of its own; the ability's duration is the attack's duration.
