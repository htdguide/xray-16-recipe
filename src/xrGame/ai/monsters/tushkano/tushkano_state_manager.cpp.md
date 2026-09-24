# src/xrGame/ai/monsters/tushkano/tushkano_state_manager.cpp

> The tushkano's whole mind: a seven-way priority ladder whose top rung is "is the thing that scares me strong or weak".

**Needs** — [`tushkano.h`](tushkano.h.md) · [`tushkano_state_manager.h`](tushkano_state_manager.h.md) · [`states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`states/monster_state_attack.h`](../states/monster_state_attack.h.md) · [`states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md) · [`states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`states/monster_state_controlled.h`](../states/monster_state_controlled.h.md) · [`states/monster_state_help_sound.h`](../states/monster_state_help_sound.h.md)
**Used by** — [`tushkano_state_manager.h`](tushkano_state_manager.h.md)
**Tier floor** — T3: a fixed priority ladder over cached perception results

## Purpose

Creature minds in this game are not planners — that is the human stalker's machinery. A
creature's mind is a **fixed priority ladder** re-evaluated from scratch every tick, with no
memory between ticks beyond what the perception components already hold. The ladder is
written once, in code, per creature, and it *is* the creature's personality.

The tushkano's ladder is the plainest example in the game, and reading it tells you exactly
what a tushkano does and in what order it cares.

## `CStateManagerTushkano` — the registered behaviours

**Contract** — construction registers eight behaviours against their identifiers. Each is a
generic composite instantiated for this creature; none is tushkano-specific.

```text
resting · attacking · eating · reacting to a dangerous sound
panicking · reacting to being hit · under psychic control · answering a call for help
```

## `execute` — the per-tick selection

**Contract** — chooses one behaviour identifier, activates it (the framework runs the
previous behaviour's teardown if the choice changed), runs it, and records the chosen
sub-state for the next tick's comparison. Allocates nothing; called on the creature's
scheduled update, so its cadence degrades with distance from the player.

```text
FUNCTION execute()
  IF under psychic control
    chosen = controlled
  ELSE
    enemy = perception.enemy.selected

    IF enemy EXISTS
      # the whole of the tushkano's character is this one branch. It does not decide
      # whether to fight by its own health or by numbers; it asks the enemy manager
      # how dangerous this enemy is *rated*, and that rating comes from configuration.
      SELECT perception.enemy.danger_rating(enemy)
        strong: chosen = panicking          # flee, look for the way out
        weak:   chosen = attacking
        # note: no default. An enemy with neither rating leaves `chosen` unset
        # and the selection falls through to the framework's invalid identifier.

    ELSE IF perception.hits.was_hit
      chosen = reacting to being hit

    ELSE IF the help-sound behaviour reports it has something to respond to
      chosen = answering a call for help

    ELSE IF heard an interesting sound OR heard a dangerous sound
      chosen = reacting to a dangerous sound

    ELSE IF there is a corpse worth eating
      chosen = eating

    ELSE
      chosen = resting

  select(chosen)
  current_behaviour.execute()
  previous_substate = current_substate
```

**Invariants**

- **The ladder is total only by luck.** Every rung but the enemy rung ends in an
  unconditional fallback to resting, so a tushkano always has something to do — *unless* an
  enemy exists whose danger rating is neither strong nor weak, in which case no branch
  assigns a behaviour. A rebuild should make the rating exhaustive or give the enemy rung a
  fallback.
- **An interesting sound and a dangerous sound select the same behaviour.** The tushkano
  does not investigate curiosities; both kinds of sound route to the danger reaction. That
  is a deliberate simplification for a prey animal and it is what distinguishes the
  tushkano's ladder from, say, the dog's.
- **Psychic control short-circuits everything**, including being hit. A controlled tushkano
  will not react to damage.

## Notes

**A call whose result is discarded.** The selection asks the corpse-perception component for
the current corpse and throws the answer away; the variable that would have held it is
commented out. The eating decision is made a few lines later through a different predicate
that consults the same component. The call is a leftover, and a rebuild should not assume
it has a side effect worth preserving — though a rebuilder should confirm, against a running
original, that the corpse query is genuinely pure, since a query that *selects* a corpse as
a side effect would make this line load-bearing.

**No cover, no grouping, no home point.** The tushkano's ladder omits every social and
positional behaviour the larger creatures have. It is the cheapest mind in the game and it
is what makes a swarm of tushkano affordable.
