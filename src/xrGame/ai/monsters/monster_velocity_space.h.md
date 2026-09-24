# src/xrGame/ai/monsters/monster_velocity_space.h

> The named movement gaits, as bit flags that combine into the gait sets a creature's states request.

**Needs** — _(none)_
**Used by** — [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`base_monster_think.cpp`](basemonster/base_monster_think.cpp.md) · [`bloodsucker.cpp`](bloodsucker/bloodsucker.cpp.md) · [`boar.cpp`](boar/boar.cpp.md) · [`burer.cpp`](burer/burer.cpp.md) · [`cat.cpp`](cat/cat.cpp.md) · [`chimera.cpp`](chimera/chimera.cpp.md) · [`control_animation_base.cpp`](control_animation_base.cpp.md) · [`control_animation_base_accel.cpp`](control_animation_base_accel.cpp.md) · [`control_animation_base_update.cpp`](control_animation_base_update.cpp.md) · [`control_jump.cpp`](control_jump.cpp.md) · [`control_movement_base.cpp`](control_movement_base.cpp.md) · [`control_rotation_jump.cpp`](control_rotation_jump.cpp.md) · _and 15 more_
**Tier floor** — T3: bit flags and their unions

## Purpose

A creature does not move at a speed; it moves in a *gait*, and each gait carries both a linear
and an angular velocity loaded from the creature's configuration section. This file names the
gaits and — more importantly — names the **sets** of gaits a behaviour may use.

The distinction is the whole point. A state does not say "run"; it says "attack at normal
health", which is the set `{ turn, walk, run }`, and the movement layer picks among them by
how far there is to go and how sharp the turn is. A state that wanted only running would
produce a creature that cannot make a tight corner.

Because the sets are unions, the values are bit flags and a set is one integer.

## The gaits

```text
ENUM Gait                  # single bits
  idle                     # standing still
  turn_in_place
  walk
  run
  walk_damaged             # the limping variants, substituted when the creature is hurt
  run_damaged
  sneak                    # the "steal" gait: slow and quiet
  drag                     # moving while hauling a corpse
  invisible                # the gait used while hidden — usually much slower
  run_attack               # the committed charge
  walk_sniffing
  walk_growling
  species_extension_base   # species add their own gaits above this
```

## The sets

```text
walk_set              = turn_in_place | walk
walk_damaged_set      = turn_in_place | walk_damaged
run_set               = turn_in_place | walk | run
run_damaged_set       = turn_in_place | walk_damaged | run_damaged
attack_set            = turn_in_place | walk | run            # identical to run_set
attack_damaged_set    = turn_in_place | walk_damaged | run_damaged   # identical to run_damaged_set
sneak_set             = turn_in_place | sneak
drag_set              = turn_in_place | drag
invisible_set         = turn_in_place | invisible
run_attack_set        = turn_in_place | run_attack
walk_growling_set     = turn_in_place | walk_growling
walk_sniffing_set     = turn_in_place | walk_sniffing
```

**Invariants** — every set includes `turn_in_place`, without exception. A creature must always
be able to turn on the spot regardless of which gaits it is allowed, or it cannot face a
target it is already standing next to.

The "run" sets include walking, so a creature allowed to run may still walk the last stretch;
the reverse is not true. The damaged sets never include the healthy gaits, so a hurt creature
cannot accidentally sprint.

**Notes** — the attack pair is byte-for-byte identical to the run pair. They are named
separately so the two behaviours can diverge in data without touching the states that request
them; nothing in the shipped configuration makes them differ.

The species extension base collides with `walk_growling` — both are bit 12. A species that
takes the extension base and a creature that uses the growling gait cannot be the same
creature, which happens to be true of every shipped species, so the collision is latent. A
rebuild should move the base up; this is recorded because the collision is not obvious from the
declaration and a new species would trip on it.

Four species declare extensions above the base — a leaping predator, a leaping humanoid, an
invisible stalker and a giant each add a jump-launch gait, and the psi-controller adds forward
and backward deliberate paces with their own sets. Each is a bit or two above the base, and
they *overlap between species*: two different species both use base+2. That is safe only
because a gait value is never interpreted outside the creature that defined it. A rebuild that
pools gait values globally must renumber.
