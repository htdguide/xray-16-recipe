# src/xrGame/ai/monsters/rats/ai_rat.h

> Declares the rat: the one creature in the game that is not built on the shared monster base, that is edible, and whose behaviour is a stack of states rather than a tree.

**Needs** — [`ai_rat.cpp`](ai_rat.cpp.md) · [`ai_rat_inline.h`](ai_rat_inline.h.md) · [`../../../CustomMonster.h`](../../../CustomMonster.h.md) · [`../../../eatable_item.h`](../../../eatable_item.h.md) · [`../../../group_hierarchy_holder.h`](../../../group_hierarchy_holder.h.md) · [`../../../rat_state_manager.h`](../../../rat_state_manager.h.md)
**Used by** — [`ai_rat.cpp`](ai_rat.cpp.md) · [`ai_rat_animations.cpp`](ai_rat_animations.cpp.md) · [`ai_rat_behaviour.cpp`](ai_rat_behaviour.cpp.md) · [`ai_rat_feel.cpp`](ai_rat_feel.cpp.md) · [`ai_rat_fire.cpp`](ai_rat_fire.cpp.md) · [`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md) · [`ai_rat_impl.h`](ai_rat_impl.h.md) · [`ai_rat_inline.h`](ai_rat_inline.h.md) · [`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) · [`rat_state_activation.cpp`](rat_state_activation.cpp.md) · [`rat_state_initialize.cpp`](rat_state_initialize.cpp.md) · [`rat_state_switch.cpp`](rat_state_switch.cpp.md) · [`rat_states.cpp`](../../../rat_states.cpp.md)
**Tier floor** — T2: a creature record with a physics body, a network image and a flat parameter block

## Purpose

The rat predates the rest of chapter 24 and never joined it. Four structural facts follow from
that, and they are the reason its files look nothing like any other creature's:

1. **It descends from the generic custom-monster base, not from the shared monster base.** It
   has no animation-action table, no motion-control abilities, no cover manager, no melee
   checker, no squad goals. Everything those give other creatures, the rat does by hand.
2. **It is simultaneously a creature and an item.** A rat corpse is a thing other creatures eat
   and the player can pick up, so the type inherits the edible-item facet as well, and nearly
   every lifecycle call in [`ai_rat.cpp`](ai_rat.cpp.md) has to be dispatched to both parents in
   the right order.
3. **Its brain is a stack, not a tree.** States push and pop rather than nest, so "go back to
   what I was doing" is a real operation the rat has and no other creature does.
4. **It steers itself.** Other creatures hand a destination to the path builder; the rat
   integrates its own heading and position every frame against the navigation mesh and backs
   out when it would leave it. See [`ai_rat_templates.cpp`](ai_rat_templates.cpp.md).

The header is also where the rat's entire tuning surface lives, which is why this page carries
the state record rather than deferring it.

## State

A rat's identity is its place in the team / squad / group hierarchy; the group record holds the
shared counters its activity budget is enforced against (see
[`ai_rat_impl.h`](ai_rat_impl.h.md)). Everything else it owns falls into three groups.

Authored in the creature's configuration section — shared by every rat in the game:

```text
RECORD RatTuning
  active_probability     : real    # chance per goal change of staying active rather than settling
  active_count_percent   : int     # what fraction of a living group may be active at once
  standing_count_percent : int     # what fraction may be standing still at once
  passive_schedule       : { min, max }   # update interval while passive: rarely
  active_schedule        : { min, max }   # update interval while active: taken from the base's own
  lost_memory_time       : int     # how long a vanished enemy is still chased
  lost_recoil_time       : int     # how long a startle lasts
  retreat_time           : int     # how long after last seeing an enemy a retreat may end
  under_fire_distance    : real    # how far to flee when frightened
  retreat_distance       : real    # how far to back off from an enemy
  stable_distance        : real    # inside this of home, the rat may choose to settle
  angle_speed            : real    # turn rate while wandering
  angular_speeds         : { still, min, max, attack }   # turn rate per movement speed
  goal_change_delta      : real    # seconds between wander-goal re-rolls
  goal_variation         : vector  # the box a wander goal is sampled from, around home
  sound_threshold        : real    # below this loudness a sound is not heard at all
  morale_quanta          : { success_attack, death, fear, restore }
  morale_restore_period  : int
  morale_bounds          : { min, max, normal }
  morale_death_distance  : real    # radius of the death broadcast
  eat_corpse_interval    : int     # how long after death a corpse is still worth eating
  eat_member_corpses     : bool    # will it eat its own team's dead
  cannibalism            : bool    # will it eat its own kind
  action_refresh_rate    : int     # how often the shared attack/retreat lottery is re-rolled
  attack_success_probability : real
  wall_turn_range        : { min, max }   # authored in degrees, stored in radians
  corpse_mass            : real
  max_health             : real

```

Authored in the spawn record — per instance, so one level's rats can be deadlier than another's:

```text
RECORD RatSpawnParameters
  speeds                 : { min, max, attack }
  eye                    : { field_of_view, range }
  hit                    : { power, interval }
  attack                 : { distance, angle }
  pursuit_radius         : real   # how far from home an enemy may be and still be chased
  home_radius            : real   # how far from home the rat may stray

```

Runtime:

```text
RECORD RatRuntime
  state_stack            : stack<StateId>      # the brain; push to divert, pop to resume
  current_state          : StateId
  morale                 : real
  home_position          : vector             # the group's shared anchor, not the spawn point
  current_graph_point    : int                 # coarse cross-level vertex the home is on
  next_graph_point       : int                 # the one it is migrating to
  graph_point_change_at  : int                 # when the migration is due
  goal_direction         : vector             # the point currently being steered toward
  goal_change_time       : real                # counts down; reaching zero re-rolls the goal
  heading_pitch_bank     : vector             # the rat's own integrated orientation
  heading_rate           : real                # its integrated turn rate, damped
  speed, safe_speed      : real                # current, and the one to return to
  no_way                 : bool                # last step was refused by the navigation mesh
  old_position           : vector             # the last accepted position, to back out to
  last_sound             : { kind, time, power, position, source }
  is_active, is_standing : bool                # membership in the two group budgets
  patrol_path            : optional<path>      # authored per-spawn; overrides wandering entirely
  current_way_point      : int
```

**Invariants**

- `is_active` and `is_standing` are not just flags: each corresponds to a counter on the shared
  group record, and setting or clearing one must increment or decrement that counter. The
  counters are what cap how many rats in a nest move at once.
- `home_position` is taken from the *squad leader*, not from the rat's own spawn point, and is
  propagated to every follower every tick. A nest therefore has one anchor and moves as one.
- `safe_speed` shadows `speed` so that a step refused by the navigation mesh can be undone
  without losing the speed the rat had chosen.
- `wall_turn_range` and the four angular speeds are authored in degrees and stored in radians.

## Where the implementation lives

The rat is split across six compiled files and two inline headers, along lines that are partly
meaningful and partly historical:

| File | What it holds |
|---|---|
| [`ai_rat.cpp`](ai_rat.cpp.md) | lifecycle: construction, load, spawn, destroy, network image, the dual-parent dispatch |
| [`ai_rat_behaviour.cpp`](ai_rat_behaviour.cpp.md) | the think step and the migration of the group's home anchor |
| [`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) | the self-steering: heading integration, speed selection, the navigation-mesh veto, patrol following |
| [`ai_rat_fire.cpp`](ai_rat_fire.cpp.md) | the bite, the hit reaction, morale, and how a corpse is valued as food |
| [`ai_rat_feel.cpp`](ai_rat_feel.cpp.md) | what the rat may see, and what a heard sound does to its morale |
| [`ai_rat_animations.cpp`](ai_rat_animations.cpp.md) | the clip table and the animation chosen from speed and turn |
| [`ai_rat_impl.h`](ai_rat_impl.h.md) | the group activity budget |
| [`ai_rat_inline.h`](ai_rat_inline.h.md) | the wander goal and the speed lottery |
| [`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md) | **dead**: the previous brain, excluded from the build and no longer compilable |

The predicates and actions the live brain calls are in
[`rat_state_switch.cpp`](rat_state_switch.cpp.md),
[`rat_state_activation.cpp`](rat_state_activation.cpp.md) and
[`rat_state_initialize.cpp`](rat_state_initialize.cpp.md); the states that call them are outside
this directory.

## The state set

```text
ENUM RatState
  death           # corpse; waits for its food value to run out, then removes itself
  free_active     # wandering near home
  free_passive    # settled; the low-update state that makes a large nest affordable
  attack_range    # in position and facing: bite
  attack_melee    # closing on the enemy
  under_fire      # frightened: run away from where I am facing
  retreat         # back off from the enemy while still watching it
  pursuit         # chase a remembered position
  free_recoil     # startled by a sound: bolt
  return_home     # too far from the nest
  eat_corpse      # feeding
  no_way          # following an authored patrol path instead of wandering
```

**Notes** — the enumeration names are historical and two of them mislead: `attack_range` is the
*melee bite* (the rat has no ranged attack) and `attack_melee` is the *approach*. The dead
brain in [`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md) still uses the older names for the same values,
which is one of several reasons it no longer compiles. `no_way` is also misnamed: it is the
patrol-following state, not a failure state.

`death` occupies value zero, which matters because a rat that is killed has its state
overwritten directly rather than pushed, and the stack is not unwound.
