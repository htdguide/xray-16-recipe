# src/xrGame/ai/monsters/cat/cat.cpp

> The cat's animation table and its unfinished pounce.

**Needs** — [`cat.h`](cat.h.md) · [`cat_state_manager.h`](cat_state_manager.h.md) · [`base_monster.h`](../basemonster/base_monster.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md)
**Used by** — [`cat.h`](cat.h.md)
**Tier floor** — T2: a creature class; the only low-level reach is resolving clip names against the model

## Purpose

The cat is the plainest creature in the chapter. Almost the whole file is the same animation table every creature declares — see [`boar.cpp`](../boar/boar.cpp.md) for what the table's columns mean, stated once. What is worth recording here is what the cat has that the boar does not, and what it *almost* has.

## State

Stateless beyond the base creature.

## `Load`

**Contract** — Reads the configuration section, declares the animation table, the posture transitions and the action bindings. Same shape and same failure mode as every creature's.

Differences from the common table worth naming:

- The cat substitutes a damaged variant for *standing idle* as well as for walk and run. Most creatures substitute only the moving motions; the cat visibly favours a limb when it stands still.
- Its acceleration chains are only walk→run and damaged-walk→damaged-run: the cat has no turning run variants, so it cannot ramp into a turn.
- It has both lying and standing postures with transitions in both directions, and no sleep motion of its own — sleeping, resting and lying idle all resolve to the same lying idle clip.
- Two spin clips (`stand_jump_ls_`, `stand_jump_rs_`) are declared but bound to no action and used by no transition: they are reachable only from the retired jump-turn path described below.

## `reinit`

**Contract** — Resolves the three clips of the pounce — its launch, its flight and its landing — and asserts each exists. Then does nothing with them.

```text
FUNCTION reinit()
  base.reinit()
  launch  = resolve_clip("jump_attack_0")   ASSERT valid
  flight  = resolve_clip("jump_attack_1")   ASSERT valid
  landing = resolve_clip("jump_attack_2")   ASSERT valid
  # register_jump_ability(launch, flight, landing)  -- disabled
```

**Notes** — The registration call is present and commented out. The assertions remain live, which means the shipped data must still carry all three clips for a cat to spawn at all, even though nothing will ever play them. A rebuild is free to drop the pounce entirely, but must then also drop the requirement on the data — or keep both, to stay bug-compatible with mods that ship their own cat models.

## `try_to_jump`

**Contract** — The pounce trigger. Fetches the current enemy, returns if there is none or if it is not visible right now, and then returns anyway. No side effects on any path.

**Notes** — This is dead code with its guards intact, which is the useful part: it records what the pounce *would* have required — a live enemy and a clear line of sight at the moment of launch, not a remembered position.

## `HitEntityInJump`

**Contract** — Applies the pounce's damage to a target. Looks up the attack-parameters row keyed by the landing clip's name and deals that row's damage, impulse and impulse direction. Unreachable in the shipped build, since nothing launches a pounce.

**Notes** — The damage is authored in configuration, keyed by *clip name*, not by creature or by action. That indirection is the engine's general rule for melee: an attack's numbers belong to the animation that delivers it, so the frame the damage lands on and the damage itself are authored together. See [`control_animation_base.cpp`](../control_animation_base.cpp.md).

## `CheckSpecParams`

**Contract** — Handles the special-parameter flags an animation can raise.

```text
FUNCTION CheckSpecParams(flags)
  IF flags contains check_corpse
    run_one_shot_sequence(clip_for(check_corpse_motion))
  IF flags contains rotation_jump
    # retired; see Notes
```

**Notes** — The corpse-inspection branch is the only live one: when the animation layer signals that the creature has reached the point of a clip where it would sniff a body, the creature queues the inspection clip as a one-shot sequence. The jump-turn branch is retired in the same way as the boar's, and its retired body is the same computation described in [`boar.cpp`](../boar/boar.cpp.md) — choose a spin clip by which side the enemy is on, overshoot the target heading slightly, then force the angular speed so the spin finishes exactly with the clip. The cat's version differs in one number: it does not multiply the derived angular speed by two and a half as the boar's does. Whether that factor was a boar-specific tuning or a fix the cat never received is not recoverable from the source.

## `UpdateCL`

**Contract** — Pure delegation to the base creature's frame update. The cat adds nothing per frame.
