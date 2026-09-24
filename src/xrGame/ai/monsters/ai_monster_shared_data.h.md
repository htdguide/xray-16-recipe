# src/xrGame/ai/monsters/ai_monster_shared_data.h

> The block of tuned numbers every creature reads from its configuration section — the data half of "creatures differ by data, not code".

**Needs** — [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md)
**Tier floor** — T3: a record of scalars loaded from text

## Purpose

This is the shared settings block of the creature base. It is short, it is pure data, and
it is one of the most important pages in the chapter: it is the concrete answer to "what
actually differs between a dog and a flesh". Nearly everything here is a threshold or a
period that shapes behaviour without changing its structure.

It is a separate header because the base creature, the animation layer and several
managers all read it, and because a settings block that lives in one place can be filled by
one loader.

## State

```text
RECORD MonsterSettings

  # --- corpses and feeding ---------------------------------------------
  corpse_search_dist   : real    # how far the creature will go to investigate a corpse
  satiety_threshold    : real    # hunger below which eating becomes attractive
  eat_frequency        : real    # seconds between bites
  eat_slice            : real    # how much of a corpse one bite consumes
  eat_slice_weight     : real    # how much satiety one slice restores

  # --- damage ----------------------------------------------------------
  damaged_threshold    : real    # health fraction below which the creature is "damaged":
                                 #   movement animations are substituted wholesale

  # --- sound pacing ----------------------------------------------------
  idle_sound_delay     : int (ms)
  eat_sound_delay      : int (ms)
  attack_sound_delay   : int (ms)
  distant_idle_delay   : int (ms)   # a separate, slower idle call heard from far away
  distant_idle_range   : real       # the distance beyond which the distant call is used

  # --- perception ------------------------------------------------------
  sound_threshold      : real    # loudness below which a heard sound is ignored
  max_hear_dist        : real

  # --- schedule --------------------------------------------------------
  day_begin            : int     # in-game hours bounding the creature's active period;
  day_end              : int     #   nocturnal creatures invert them

  # --- body ------------------------------------------------------------
  legs                 : int     # 4 or 2; selects footstep grouping
  attack_effector      : AttackEffector   # the screen effect this creature's blow causes

  # --- running attack --------------------------------------------------
  run_attack_path_dist : real    # how long a path the run-attack plans
  run_attack_start_dist: real    # range at which a run-attack may be launched
```

**Invariants** — the two sound delays and the distant pair describe one mechanism between
them: a creature idling near the player uses the close call at the close period, and one
beyond the distant range uses the distant call at the distant period. The ranges must not
be contradictory, and nothing checks them.

`damaged_threshold` is the trigger for the animation substitution table described in
[`ai_monster_defs.h`](ai_monster_defs.h.md): crossing it swaps every movement animation for
its damaged variant, which is the most visible single data-driven behaviour change in the
chapter.

The day window is in in-game hours and is what makes some creatures nocturnal. It is read
by the rest state, not by anything here.

**Notes** — the leading comment in the original announces "float speed factors" and is
followed by no speed factors; they moved elsewhere and the comment did not. There is no
loader in this file — each creature's startup fills the block from its own section, which
is why the field names and the configuration key names are not always the same and must be
read off the loader rather than guessed from here.
