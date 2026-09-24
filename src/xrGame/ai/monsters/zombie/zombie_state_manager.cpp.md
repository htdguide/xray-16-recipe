# src/xrGame/ai/monsters/zombie/zombie_state_manager.cpp

> The zombie's mind, defined as much by what it omits as by what it has: no panic, no flight, no reaction to being hit.

**Needs** — [`zombie.h`](zombie.h.md) · [`zombie_state_manager.h`](zombie_state_manager.h.md) · [`zombie_state_attack_run.h`](zombie_state_attack_run.h.md) · [`states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`states/monster_state_attack.h`](../states/monster_state_attack.h.md) · [`states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`states/monster_state_controlled.h`](../states/monster_state_controlled.h.md) · [`states/monster_state_help_sound.h`](../states/monster_state_help_sound.h.md)
**Used by** — [`zombie_state_manager.h`](zombie_state_manager.h.md)
**Tier floor** — T3: a five-rung priority ladder plus one suppression check

## Purpose

Compare this ladder with the tushkano's and the zombie's character falls straight out of the
diff. The tushkano has panic, a hit reaction and a dangerous-sound reaction; the zombie has
none of the three. It cannot be frightened, it does not flinch, and it does not distinguish
a dangerous noise from an interesting one because it has no danger response to route to.
It sees an enemy and it attacks. That is the creature.

The one thing above its ladder is the feigned-death suppression.

## `CStateManagerZombie` — the registered behaviours

**Contract** — construction registers six behaviours. Five are generic; the attack is
**composed** rather than taken whole:

```text
resting · eating · reacting to an interesting sound
under psychic control · answering a call for help
attacking = generic attack composite, given
              approach phase = the zombie's own shambling approach
              strike  phase = the generic melee strike
```

That composition is the pattern the creature layer uses wherever a creature needs one phase
of a standard behaviour changed: keep the composite, replace a sub-state. The zombie
replaces only the approach, because how a zombie *closes* is its second-most recognisable
trait after feigning death. See [`zombie_state_attack_run.h`](zombie_state_attack_run.h.md).

## `execute` — the per-tick selection

**Contract** — refuses to run at all while any animation triple is active, then walks a
five-rung ladder.

```text
FUNCTION execute()
  # the suppression. A feigned death, or any other three-phase animation the zombie
  # is running, freezes the mind entirely: no behaviour is selected and no behaviour
  # is executed, so the zombie does not path, does not turn and does not vocalise
  # until the animation finishes. This is why feigned death did not need to be a state.
  IF an animation triple is active THEN RETURN

  IF under psychic control
    chosen = controlled
  ELSE IF an enemy is selected
    chosen = attacking              # unconditional: no danger rating, no health test
  ELSE IF the help-sound behaviour has something to respond to
    chosen = answering a call for help
  ELSE IF heard an interesting sound OR heard a dangerous sound
    chosen = reacting to an interesting sound
  ELSE IF there is a corpse worth eating
    chosen = eating
  ELSE
    chosen = resting

  select(chosen)
  current_behaviour.execute()
  previous_substate = current_substate
```

**Invariants**

- The ladder is total: every rung falls through to resting.
- Both kinds of sound route to the *interesting* sound behaviour — investigate, not hide.
  The zombie walks toward noises.
- Suppression returns before `previous_substate` is updated, so the record of what the
  zombie was doing survives the feigned death and the behaviour that resumes afterwards
  sees an unchanged history. That is what lets a zombie get up and carry on attacking
  rather than re-entering its attack from the beginning.

## Notes

**No hit reaction is registered**, so the animation-modifier and stagger effects declared on
the zombie's motions are driven by the base creature's hit handling rather than by a
behaviour. The zombie visibly recoils but never *decides* anything about having been shot —
except, in the hit handler and not here, to fall over.
