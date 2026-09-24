# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_hide_inline.h

> After a feed the creature sprints away from where the player is, and only once it has broken contact does it resume stalking.

**Needs** — [`bloodsucker_vampire_hide.h`](bloodsucker_vampire_hide.h.md) · [`state_hide_from_point.h`](../states/state_hide_from_point.h.md) · [`bloodsucker_predator.h`](bloodsucker_predator.h.md) · [`state.h`](../state.h.md)
**Used by** — [`bloodsucker_vampire_hide.h`](bloodsucker_vampire_hide.h.md)
**Tier floor** — T3: pure selection and parameter authoring over states written elsewhere

## Purpose

The escape half of the vampire behaviour. It exists as its own node because the retreat is two different behaviours in sequence — a panic sprint and a patient stalk — and the tree above wants to treat that pair as one thing it can re-enter.

## State

Stateless; the two substates and the "which substate ran last" marker are the shared machinery's.

```text
SUBSTATES
  run_away : generic "flee from a point" state, parameterised below
  predator : the bloodsucker's stalking state, shared with its ordinary attack
```

## `VampireHideState` construction

**Contract** — Registers the two substates. Allocates both eagerly; they live as long as the creature.

## `reselect_state`

**Contract** — Chooses the substate for the coming tick. Runs only when no substate is active.

```text
FUNCTION reselect_state()
  IF previous_substate == run_away AND predator.check_start_conditions()
    select(predator)
    RETURN
  select(run_away)
```

**Notes** — The sprint is the default and the stalk is the reward for having finished one: the creature will keep re-entering the sprint until the stalking state agrees the conditions are right, which is what makes a fed bloodsucker look like it panics first and schemes second.

## `setup_substates`

**Contract** — Fills the generic flee state's parameter record at the moment it is selected. Reads the last known player position and the creature's configured attack-sound spacing; writes nothing else.

```text
FUNCTION setup_substates()
  IF current_substate == run_away
    flee.point        = last_known_enemy_position
    flee.distance     = 50 world units      # how far away counts as "away"
    flee.accelerated  = true
    flee.braking      = false
    flee.accel_type   = aggressive
    flee.action       = run
    flee.sound_type   = aggressive
    flee.sound_delay  = configured attack-sound spacing for this creature
    flee.time_out     = 15000 ms            # give up fleeing after this
```

**Notes** — The flee distance and the timeout are literals here, not configuration: fifty units is roughly "out of the room and down the corridor" on the shipped levels, and the fifteen-second cap is what stops a creature cornered against geometry from sprinting into a wall forever. The parent [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md) authors the identical record for its own copy of the flee state, and the two must stay in step.

## `check_completion`

**Contract** — The whole retreat is finished when the *stalking* substate finishes. Finishing the sprint alone never ends it, which is what forces the sequence sprint-then-stalk rather than sprint-then-leave.
