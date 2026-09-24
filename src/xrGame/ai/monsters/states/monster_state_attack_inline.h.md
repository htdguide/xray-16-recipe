# src/xrGame/ai/monsters/states/monster_state_attack_inline.h

> How a creature fights: an ordered chain of eight alternatives, tried top to bottom every tick, each of which is either "continue what I was doing" or "start something new", with melee as the fallback when none of them applies.

**Needs** — [`monster_state_attack.h`](monster_state_attack.h.md) · [`monster_state_attack_melee.h`](monster_state_attack_melee.h.md) · [`monster_state_attack_run.h`](monster_state_attack_run.h.md) · [`monster_state_attack_run_attack.h`](monster_state_attack_run_attack.h.md) · [`monster_state_attack_on_run.h`](monster_state_attack_on_run.h.md) · [`monster_state_attack_camp.h`](monster_state_attack_camp.h.md) · [`state_hide_from_point.h`](state_hide_from_point.h.md) · [`monster_state_find_enemy.h`](monster_state_find_enemy.h.md) · [`monster_state_steal.h`](monster_state_steal.h.md) · [`monster_state_home_point_attack.h`](monster_state_home_point_attack.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../../../Actor.h`](../../../Actor.h.md)
**Used by** — [`monster_state_attack.h`](monster_state_attack.h.md) · [`monster_state_controlled_attack_inline.h`](monster_state_controlled_attack_inline.h.md)
**Tier floor** — T3: a priority chain over registered child states

## Purpose

This is the most reused page in chapter 24: almost every creature's combat is this file, and
the creature contributes only the numbers its melee checker and its configuration section
supply, plus two predicates — can it run-attack, and can it attack while moving.

Three ideas carry it.

**The chain is not a state machine, it is a re-evaluated priority list.** There are no
transitions. Every tick, the eight alternatives are tried in a fixed order and the first that
applies wins, so a creature can be pulled out of any behaviour at any moment by a
higher-priority one becoming true. Continuity is achieved *within* each test, not by the chain.

**Every test has the same two-branch shape**: if I was already doing this, keep doing it until
it reports completion; otherwise, ask whether it may start. That idiom — `previous_child == X`
against `check_completion`, versus `check_start_conditions` — is what makes a chain with no
transitions behave like one with them, and it is worth reading once here rather than in each
test.

**The fallback is melee versus approach, and it is the only branch with hysteresis.**

## Construction

**Contract** — registers nine children: approach, melee, run-attack, attack-on-run, flee to
cover, search for a lost enemy, stalk, camp in cover, and move to the authored home point.
The second constructor takes the approach and melee states from the caller, so a creature with
a distinctive charge or bite substitutes just those two and inherits the rest of the chain.
Allocates one instance of every child regardless of whether the creature can ever use it.

**Notes** — the two constructors differ in exactly two registrations and duplicate the other
seven. A rebuild should have one constructor taking optional overrides.

## `initialize`

**Contract** — entered when the brain selects combat. Arms the creature's melee checker for a
fresh attack — which resets its hysteresis, so the minimum and maximum bite distances start from
their base values — and clears the three timers.

## `execute` — the chain

**Contract** — chooses a child, runs it, records it as previous, and publishes the creature's
goal to its squad. Called every tick.

```text
FUNCTION execute()
  can_attack_on_move = creature.can_attack_on_move()

  IF check_home_point()                            child = move_to_home_point
  ELSE IF check_steal_state()                      child = stalk
  ELSE IF check_camp_state()                       child = camp_in_cover
  ELSE IF check_find_enemy_state()                 child = search_for_enemy
  ELSE IF check_run_away_state()                   child = flee_to_cover
  ELSE IF NOT can_attack_on_move AND check_run_attack_state()
                                                   child = run_attack
  ELSE IF can_attack_on_move                       child = attack_on_run
  ELSE
    # the fallback pair, with hysteresis
    IF previous_child == melee AND NOT melee.check_completion()
      child = melee                                # stay in melee until the checker releases
    ELSE IF melee.check_start_conditions()
      child = melee
    ELSE
      child = approach

  select_state(child)
  current_child.execute()
  previous_child = current_child

  publish to my squad: goal = attack, target = my enemy
```

**Invariants**

- **The ordering is the design.** Returning to an authored home point outranks everything,
  including fighting — a creature dragged too far from its post abandons the fight. Stalking
  outranks camping, camping outranks searching, searching outranks fleeing, and fleeing outranks
  any form of attack. So a creature that has lost sight of its enemy stops fighting to look for
  it, and one that is despondent stops looking to run.
- **`can_attack_on_move` splits the chain in two.** A creature that can attack while moving
  never plays the run-attack and never falls through to the melee/approach pair at all: it goes
  straight to attack-on-run and stays there, because that state never reports completion. So for
  such creatures the last four branches collapse to one. Creatures that cannot get the classic
  approach-and-bite.
- **The squad publication happens unconditionally, after the child ran**, so squadmates see this
  creature's target even on ticks where it was fleeing or searching. That is what keeps a pack
  converging on one enemy.
- **Nothing here checks that an enemy exists.** The brain above guarantees it, and several of
  the tests dereference it. A rebuild should make it explicit.

## `check_home_point`

**Contract** — two-branch idiom. If the creature was not already returning home, ask whether the
home-point state wants to start; if it was, keep going until it completes. The home-point state
owns the actual rule (how far is too far), and this is only the continuity wrapper.

## `check_steal_state` and `check_camp_state`

**Contract** — the same idiom with one difference: the start branch is taken **only when no
child has ever run** (`previous_child` is unset), not merely when the previous child was
something else.

**Invariants** — that restriction is load-bearing and easy to miss. Stalking and camping can
only begin on the *first* tick of a fight. A creature that has already closed with its enemy and
then loses the initiative cannot drop back into a stalk or a camp; it must leave combat entirely
and re-enter. Which is why ambushing creatures ambush once and then commit.

## `check_find_enemy_state`

**Contract** — true when the enemy has not been seen for twelve seconds. No continuity branch and
no start condition: it is a pure timeout, so the search begins and ends purely on whether the
enemy has been seen recently.

**Notes** — twelve seconds is compiled in and shared by every creature in the game. It is one of
the most visible numbers in the chapter — it is how long a monster keeps attacking a player who
has broken line of sight — and nothing records why twelve.

## `check_run_away_state`

**Contract** — whether the creature should flee to cover.

```text
FUNCTION check_run_away_state() -> bool
  IF behinder_started != 0                RETURN false      # permanently false; see the header

  IF previous_child == flee_to_cover
    IF NOT flee_to_cover.check_completion()  RETURN true    # keep fleeing
    next_run_away_allowed_at = now + 10 seconds             # completed: start the cooldown
    RETURN false

  IF my enemy is NOT the player
     AND my morale is despondent
     AND now > next_run_away_allowed_at
    RETURN true

  RETURN false
```

**Invariants**

- **A creature never flees from the player.** The enemy must be something other than the player
  for the flight to be considered at all. Monsters break and run from each other, and from
  non-player humans, but they always stand and fight the player. That single clause shapes the
  whole feel of the game's combat and a rebuild that drops it makes every fight a chase.
- **The ten-second cooldown is armed on completion, not on entry**, so a creature that flees
  successfully will not flee again for ten seconds, while one whose flight is interrupted by a
  higher branch is free to resume immediately.

## `check_run_attack_state`

**Contract** — whether to play the charging attack. Refused outright by creatures whose
ability predicate says they have none. Otherwise: the charge may *begin* only from the approach
state, and once begun continues until it completes.

**Invariants** — restricting the start to "I was approaching" means a creature cannot charge out
of melee, out of a flight, or out of a camp. The charge is specifically the transition from
running-at to hitting-while-running.

## The melee fallback's hysteresis

**Contract** — the only branch with memory. If the creature was in melee and the melee checker
has not released it, stay; otherwise ask the checker whether melee may start; otherwise approach.

**Invariants** — the asymmetry is the point and it lives in the creature's melee checker, not
here: the distance at which melee *starts* is smaller than the distance at which it *stops*, so
a creature that has engaged does not flicker back to approaching the moment its target steps
back half a unit. Both distances are authored per creature.

## `setup_substates`

**Contract** — parameterises the flight child at the moment it is selected, and only that child.
Supplies the enemy's position as the point to hide from, a twenty-unit distance, the running
action, aggressive acceleration with braking off, the creature's authored attack-sound interval,
and a five-second timeout.

**Invariants** — the hide-from point is captured **once, at selection**, not tracked. A fleeing
creature runs away from where its enemy was when it decided to run, and does not re-evaluate.
That is what lets a monster be outflanked while retreating, and it is why the five-second
timeout exists: the flight must end on its own before the stale point becomes absurd.
