# src/xrGame/ai/monsters/chimera/chimera.cpp

> The chimera's definition: an animation table built almost entirely out of two clips, two velocity profiles for spinning and launching, and the pounce's clip triple.

**Needs** — [`chimera.h`](chimera.h.md) · [`chimera_state_manager.h`](chimera_state_manager.h.md) · [`base_monster.h`](../basemonster/base_monster.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`control_path_builder_base.h`](../control_path_builder_base.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md)
**Used by** — [`chimera.h`](chimera.h.md)
**Tier floor** — T2: a creature class

## Purpose

The chimera's table follows the pattern stated once in [`boar.cpp`](../boar/boar.cpp.md). Three things here are its own: two extra velocity profiles that the shared action mapping does not know about, the pounce's three-clip registration, and the seven attack numbers the attack state runs on.

## State

```text
RECORD Chimera EXTENDS BaseMonster
  spin_velocity   : velocity profile   # read from the section as "Velocity_Rotate"
  launch_velocity : velocity profile   # read from the section as "Velocity_JumpStart"
  attack_params   : AttackParams       # see chimera.h.md
```

## `Load`

**Contract** — Reads the section, declares the table, binds actions, reads the attack tuning. Same shape and failure mode as every creature's.

Its departures from the common table:

- **One idle clip does almost everything.** Standing idle, lying idle, sleeping, looking around, the melee attack and dying all resolve to the same standing-idle clip prefix. The chimera has no distinct clips for any of them; its attack is the pounce, so the melee "attack" motion is a placeholder that the pounce path never plays.
- **Two speeds of turn.** Besides the usual slow standing turns it declares fast left and right turns bound to the spin velocity profile. The attack state picks the fast pair whenever it must swing round to face a pounce target, because the slow turn would lose the opening.
- **The pounce's launch** is a logical motion of its own, bound to the launch velocity profile, so that the animation layer moves the creature at the launch speed while the wind-up plays.
- Substitutions and acceleration chains follow the boar's: damaged variants for walk and run, turning-run variants while turning, and chains letting walk ramp into run or either turning run.
- Walking backwards and dragging are bound to nothing: the chimera cannot drag a corpse.

```text
FUNCTION Load(section)
  base.Load(section)
  spin_velocity.load(section, "Velocity_Rotate")
  launch_velocity.load(section, "Velocity_JumpStart")
  ...declare table, transitions, action bindings...

  attack_params.attack_radius         = section value or 10 world units
  attack_params.prepare_jump_timeout  = section value or 2000 ms
  attack_params.attack_jump_timeout   = section value or 1000 ms
  attack_params.stealth_timeout       = section value or 2000 ms
  attack_params.force_attack_distance = section value or 8 world units
  attack_params.num_attack_jumps      = section value or 4
  attack_params.num_prepare_jumps     = section value or 2
  post_load(section)
```

**Notes** — Every attack number has a code default, so a section may omit all seven and still produce a working chimera. The defaults are the shipped behaviour: four damaging pounces, then two repositioning ones, then four again. `force_attack_distance` is read and stored and never read again by anything in the engine — see the Notes on [`chimera_attack_state_inline.h`](chimera_attack_state_inline.h.md).

## `reinit`

**Contract** — Loads the ground-travel velocity profile used while airborne in a pounce, and registers the pounce as a clip triple with the shared jump ability.

```text
FUNCTION reinit()
  base.reinit()
  load_velocity(section, "Velocity_JumpGround", chimera_jump_ground_profile)
  configure_jump(prepare_clip = none,
                 ground_clip  = none,
                 flight_clip  = "jump_attack_1",
                 landing_clip = "jump_attack_2",
                 travel_velocity_profile = none,
                 ground_velocity_profile = chimera_jump_ground_profile)
```

**Notes** — The first two clip slots are passed as absent. The shared jump ability supports a wind-up and a run-up clip; the chimera supplies neither, because its attack state plays the wind-up itself as an override animation and times the launch off that clip's length. Where the boar delegates its whole spin to the shared ability, the chimera keeps the timing and hands the ability only the parts that need physics.

## `CustomVelocityIndex2Action`

**Contract** — Resolves a chimera-only velocity profile to an abstract action, so the animation layer can choose a clip for a path segment authored at that speed. Both chimera profiles — the airborne ground speed and the pounce preparation speed — map to *run*. Anything else maps to standing idle.

**Notes** — This is the escape hatch in the shared velocity-to-action table: the shared table in [`control_animation_base.cpp`](../control_animation_base.cpp.md) handles the profiles every creature has and delegates the rest here.

## `HitEntityInJump`

**Contract** — Applies the pounce's damage: looks up the attack-parameters row keyed by the *flight* clip's name and deals that row's damage, impulse and impulse direction to the target.

**Notes** — Keyed on the flight clip, not the landing one — the chimera hits on contact in mid-air, which is why a pounce that is dodged deals nothing even though the landing still plays.

## `jump`

**Contract** — Launches a scripted pounce at a world position with a strength factor, and plays the aggressive vocalisation. Exists so that a level script can stage a pounce the attack state would not have chosen.

## `CheckSpecParams`

**Contract** — Implements none of the animation special-parameter flags; the body is empty. The retired branches would have played a dedicated threaten clip and a dedicated charging-attack clip, neither of which the shipped model has.

## `UpdateCL`

**Contract** — Pure delegation to the base creature's frame update.
