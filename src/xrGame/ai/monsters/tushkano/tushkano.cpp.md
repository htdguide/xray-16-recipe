# src/xrGame/ai/monsters/tushkano/tushkano.cpp

> The tushkano is a creature made entirely of an animation table: six motions, one of which is its whole attack.

**Needs** — [`tushkano.h`](tushkano.h.md) · [`tushkano_state_manager.h`](tushkano_state_manager.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md)
**Used by** — [`tushkano.h`](tushkano.h.md)
**Tier floor** — T3: a table build and two empty hooks

## Purpose

Shows what the minimum creature costs. Everything that makes a tushkano a tushkano rather
than a dog is (a) which motions its model has, (b) which actions map to which motions, and
(c) the numbers in its configuration section. There is no tushkano-specific behaviour code
at all.

The file is worth reading for the *shape* of creature construction, which every other
creature follows: install a behaviour tree, then in load, bind velocity profiles, declare
motions, and map abstract actions onto them.

## `CTushkano` — construction and destruction

**Contract** — constructs the tushkano's behaviour tree and takes ownership of it; registers
the creature with the controllable mixin so a controller psychic can enslave it. Destruction
destroys the behaviour tree. The tree is owned by the creature, not shared.

## `Load`

**Contract** — runs the base creature's load, then builds this creature's animation table
from its configuration section and finishes with the base class's post-load. Allocates. Not
thread-safe; runs once, at spawn.

```text
FUNCTION Load(section)
  base.Load(section)

  animation.load_acceleration_parameters(section)
  animation.acceleration_chain(from = walk_forward, to = run)
    # declares that walking accelerates into running; the chain is what lets the
    # creature blend speed continuously instead of snapping between two gaits

  # four velocity profiles, fetched by name from the movement component and
  # then attached to motions. The profile carries the speed the motion implies,
  # so that the mesh moves at the rate its footfalls suggest.
  idle  = movement.velocity_profile(idle)
  turn  = movement.velocity_profile(stand)
  walk  = movement.velocity_profile(walk_normal)
  run   = movement.velocity_profile(run_normal)

  # motion declarations. The second argument is a name PREFIX: the engine picks a
  # random numbered variant at play time, which is how one declaration covers
  # several authored takes of the same action.
  declare(stand_idle,       "stand_idle_",       velocity = idle, posture = standing)
  declare(stand_turn_left,  "stand_turn_left_",  velocity = turn, posture = standing)
  declare(stand_turn_right, "stand_turn_right_", velocity = turn, posture = standing)
  declare(walk_forward,     "stand_walk_fwd_",   velocity = walk, posture = standing)
  declare(run,              "stand_run_",        velocity = run,  posture = standing)
  declare(attack,           "stand_attack_",     velocity = turn, posture = standing)

  # the action-to-motion map. Thirteen abstract actions, six motions:
  # everything the tushkano cannot express collapses onto standing idle.
  link(stand_idle | sit_idle | lie_idle | eat | sleep | rest | drag | steal
       | look_around                        -> stand_idle)
  link(walk_forward | walk_backward         -> walk_forward)
  link(run                                  -> run)
  link(attack                               -> attack)

  base.post_load(section)
```

**Invariants** — the action map must be total: every action the behaviour tree can request
needs a motion, or the creature freezes. The tushkano satisfies that by collapsing eight
distinct actions onto its idle motion, which is why a tushkano told to sleep, eat or drag a
corpse simply stands there.

## `CheckSpecParams`

**Contract** — the hook through which the current state asks for an animation modifier
(inspect a corpse, look scared). **The tushkano's implementation is empty**: both branches
that would have responded are commented out, along with the motions they needed. So the
animation-modifier field that the behaviour tree faithfully sets on this creature every tick
is read by nothing. A tushkano never plays a corpse-inspection or scared animation.

## Notes

**A large disabled second table.** Roughly half the file is commented out: damaged variants
of walking and running with their own velocity profiles and an action-substitution rule that
would swap them in when the creature is hurt; sitting and standing-up transitions; eating,
looking around, stealing and dying motions. The tushkano's model presumably lacks them. A
rebuilder should not try to restore this — the shipped creature has six motions and that is
what its behaviour looks like.

**Backward walking plays the forward motion.** Not an oversight worth fixing: the tushkano
is quadrupedal and turns rather than backing up, so the behaviour tree never requests it.
