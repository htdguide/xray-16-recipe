# src/xrGame/ai/monsters/states/monster_state_steal_inline.h

> Creep up on an enemy who has not noticed you, as long as everything stays quiet and you are
> between four and fifteen units away.

**Needs** — [`monster_state_steal.h`](monster_state_steal.h.md)
**Used by** — [`monster_state_steal.h`](monster_state_steal.h.md)
**Tier floor** — T3: one predicate and a movement directive

## Purpose

The ambush leaf, and the reason a player can be killed by something that never made a sound. It is
offered first in the generic attack behaviour's cascade, so every creature with an enemy tries to
stalk before it tries anything else — and almost always fails the predicate immediately, which is
why stalking is rare and startling rather than constant.

Its whole content is one seven-clause predicate. That predicate is used **twice**: as the start
condition, and negated as the completion test. A rebuild must keep that identity, because it is
what makes the leaf abandon the stalk on exactly the conditions that would have prevented it.

## State

`Stateless.` Every clause is read from the memory components on demand.

## `check_conditions` — the seven clauses

**Contract** — true when all of the following hold.

```text
FUNCTION check_conditions() -> bool
  IF NOT enemy_memory.can_see_enemy_right_now     RETURN false   # 1
  IF enemy_memory.enemy_count > 1                 RETURN false   # 2
  IF enemy_memory.stats_are_ready                                 # 3
    IF enemy_is_moving_away_fast                  RETURN false   # 4
    IF NOT enemy_is_unaware_of_me                 RETURN false   # 5
  IF senses.heard_danger_sound                    RETURN false   # 6
  IF hit_memory.is_hit                            RETURN false   # 7
  dist = melee_range_to(enemy)
  RETURN 4 <= dist <= 15                                          # 8
```

**Notes** — each clause removes a situation in which creeping would be either impossible or
stupid, and together they describe the ambush precisely.

*Must see the enemy now, not remember it.* A stalk is a visual approach; a creature creeping toward
a remembered position is just moving slowly toward nothing.

*Exactly one enemy.* With two or more, something is already looking the other way and the surprise
is gone — and more practically, the creature would be creeping in full view of the second one.

*The two-clause block guarded by "stats are ready".* The enemy memory maintains derived facts about
a tracked enemy — whether it is pulling away quickly, whether it appears unaware — and those facts
take time to become valid after acquisition. While they are *not* ready, both clauses are skipped
and the stalk is permitted. That is a deliberate optimism: a freshly acquired enemy is assumed
unaware, which is exactly the moment an ambush is most likely to be real. Once the facts are ready
the creature checks them honestly and breaks off if the enemy is fleeing or has noticed it.

*No dangerous sound, no hit.* Both would mean the situation has already become a fight, and both
are also what a stalk would have to be abandoned for — which is why the same predicate serves as
the completion test.

*The four-to-fifteen band.* Below four the creature is already at striking range and should attack,
not creep. Above fifteen the approach would take long enough that the enemy is almost certain to
turn around, and the creature is better off charging. The distance is measured by the melee
component as its own reach-to-enemy, not as a plain centre-to-centre distance, so the lower bound
scales with the creature's size.

**Dead clause.** A further condition is declared and commented out: that the planned route's
deviation from the straight line to the enemy stay within thirty degrees. The threshold constant is
still declared and is referenced nowhere. Its intent is clear — a creeping approach that has to
loop around an obstacle is no longer a stalk — and a rebuild that wants it must also decide what to
do on the frames before the route exists, which is what the guard in the disabled code was
wrestling with. As shipped, a creature will stalk along an arbitrarily contorted route.

## `execute`

**Contract** — request the stalking action with the calm acceleration profile and no braking, path
to the enemy's remembered position and vertex with the generic parameters, and play the stealth
voice.

**Notes** — the *calm* profile on an approach to an enemy is the file's most deliberate choice and
the opposite of every other combat movement leaf, which use the aggressive profile. Accelerating
into a sprint is what gives a charge away; the stalk trades closing speed for silence.

There is a stealth voice, and it is played — a stalking creature is not silent, it is *quiet*. That
is the audio cue a player gets, and it is the only warning the leaf gives.

## `check_start_conditions` / `check_completion`

**Contract** — the predicate, and its negation.

**Notes** — writing them as exact complements means the leaf runs for precisely as long as stalking
makes sense, and hands back to the attack cascade the instant it does not — at which point the
cascade's next branches take over and the creature charges. The transition from a silent approach
to a sudden charge, which is what the player experiences, is produced entirely by this negation.
