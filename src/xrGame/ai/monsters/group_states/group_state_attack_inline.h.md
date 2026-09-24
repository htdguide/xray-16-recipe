# src/xrGame/ai/monsters/group_states/group_state_attack_inline.h

> The pack attack brain: defend the territory first; if the enemy has earned aggression, charge;
> otherwise stalk it inward through three shrinking rings and growl at it until it leaves.

**Needs** — [`group_state_attack.h`](group_state_attack.h.md) · [`group_state_attack_run.h`](group_state_attack_run.h.md) · [`group_state_squad_move_to_radius.h`](group_state_squad_move_to_radius.h.md) · [`group_state_custom.h`](group_state_custom.h.md) · [`group_state_home_point_attack.h`](group_state_home_point_attack.h.md) · [`../states/monster_state_attack_melee.h`](../states/monster_state_attack_melee.h.md) · [`../states/monster_state_attack_run_attack.h`](../states/monster_state_attack_run_attack.h.md) · [`../states/monster_state_attack_on_run.h`](../states/monster_state_attack_on_run.h.md) · [`../states/state_hide_from_point.h`](../states/state_hide_from_point.h.md) · [`../states/monster_state_find_enemy.h`](../states/monster_state_find_enemy.h.md) · [`../states/monster_state_home_point_attack.h`](../states/monster_state_home_point_attack.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`group_state_attack.h`](group_state_attack.h.md)
**Tier floor** — T3: a selector over eleven substates plus two time-based latches

## Purpose

The largest decision in this chapter, and the one that produces the series' most recognisable
creature behaviour: a pack of dogs that circles at a distance, closes in stages, growls, and only
charges when provoked. It is the pack counterpart of the generic attack composite, and it earns
its size with two things the generic one does not have — **squad coordination** (every tick it
publishes its target to the squad and reads the squad's commands) and **an aggression decision
separate from the enemy's existence**.

The separation is the key idea. Having an enemy does not mean attacking it. The creature computes,
every tick, whether it is *aggressive* toward this enemy, from the enemy's kind, where the enemy
stands relative to the pack's home region, whether gunfire has been heard, and whether the squad
has already committed. Aggressive means charge; not aggressive means the stalking ladder.

## State

```text
RECORD GroupAttackState
  enemy                     : object            # latched on entry, cleared if destroyed
  time_next_run_away        : int               # declared; never read
  time_start_check_behinder : int               # stage 1 of the behind-me tracker
  time_start_behinder       : int               # stage 2 of the behind-me tracker
  time_start_drive_out      : int               # when the current threat display began
  delta_distance            : real              # 0..3, drawn once per activation
  drive_out                 : bool              # the threat display has run long enough to escalate
```

**`delta_distance` is the file's most important field and the least obvious.** It is a random
offset in the range 0 to 3 world units, drawn once when the creature enters the attack, and it is
added to *every* ring radius in the stalking ladder. Without it, every member of a pack computes
identical ring radii and they converge into one point; with it, they form a loose arc. A rebuild
that drops it gets a pack that stacks.

## `initialize`

**Contract** — arm the melee tracker for a new attack, clear the escalation flag, latch the
enemy, draw the per-activation distance offset, reset the three clocks. If the creature is in a
squad: publish an *attack this entity* goal, and if the creature is not yet a member of the
squad's formation, make itself leader and have the squad index its members against this enemy.
Then have the squad recompute everyone's commands.

**Notes** — claiming leadership on joining is a first-come rule, not a merit rule: the creature
that notices the enemy first directs the pack. The squad's indexing is what
[`group_state_squad_move_to_radius_inline.h`](group_state_squad_move_to_radius_inline.h.md) later
uses to fan members out by index, so it must happen before any member starts stalking.

The comment in the source warns against refreshing the enemy memory here, because doing so can
invalidate the enemy the next tick reads. A rebuild should latch once and tolerate the latch going
stale, which is what `remove_links` exists for.

## `execute`

**Contract** — compute aggression, then select and run exactly one substate. Publishes the current
target to the squad at the end of every tick. Never leaves the composite without an active
substate except through the one early return noted below.

```text
FUNCTION execute()
  enemy_is_player   = enemy is the player character
  enemy_in_outer_home = home.at_home(enemy.position)
  enemy_in_mid_home   = home.at_mid_home(enemy.position)

  # a script may impose an enemy; adopt it as a real one and drop the imposition
  IF enemy is the script-imposed enemy
    adopt it as a remembered enemy; clear the imposition; re-target if needed

  # --- the aggression decision
  IF home.is_aggressive()                aggressive = true    # authored per home region
  ELSE
    aggressive = false
    IF NOT enemy_is_player               aggressive = true    # creature fights are never cautious
    IF enemy_in_mid_home AND heard_dangerous_sound  aggressive = true
    IF enemy_in_outer_home
      IF distance(self, enemy) <= 6      aggressive = true
    ELSE
      aggressive = false                 # outside the home region, nothing makes us aggressive

  IF aggressive AND squad EXISTS         squad.mark_home_in_danger()
  IF squad EXISTS AND squad.home_in_danger()  aggressive = true   # the pack commits together

  IF should_defend_home_point()          # the same latch every attack composite uses
     select move_to_home_point, or find_enemy once that completes
  ELSE IF melee is applicable
     select melee, or run_away IF the enemy has been behind us too long
  ELSE IF aggressive
     select attack_on_run IF the creature can strike while moving, else run
  ELSE
     advance the stalking ladder (below)

  IF the chosen substate is not melee   reset the behind-me tracker
  run the chosen substate
  publish the attack goal to the squad
```

**Notes on the aggression rules** — each clause is a separate authored behaviour and they are
listed in the order they override each other.

*A home region may be marked aggressive in the level data*, in which case its creatures never
stalk. That is the level designer's override and it wins outright.

*Creature-versus-creature fights skip the whole ladder.* The stalking behaviour exists to be
watched by the player; two creatures fighting each other simply fight.

*Gunfire inside the middle ring makes the pack aggressive*, which is why firing a weapon near a
pack that was circling you brings it in.

*Outside the outer home ring, aggression is forced off* — the final `else` overrides everything
above it. A pack will not charge an enemy standing outside its territory, no matter what else is
true, and this is what keeps packs anchored to their authored regions.

*Once any member is aggressive, the squad is marked and every member reads the mark.* Aggression
is therefore a pack-level latch with no decay in this file; it is cleared by the squad.

## The stalking ladder

When not aggressive, the creature walks a fixed cycle of four substates, each of which ends by
bringing it one ring closer, and the ring radii all carry the per-creature random offset.

```text
  steal          -> move to within 15 + delta of the enemy, at a run
  attack_hidden  -> move to within 10 + delta of the enemy, at a walk
  custom         -> move to within  6 + delta of the enemy, growling
                    (or within 1 if the display has already run out of patience)
  control_fire   -> stand and play the growl animation

  transitions:
    steal complete                       -> attack_hidden
    attack_hidden complete               -> custom
    attack_hidden, enemy escaped past 17 + delta -> back to steal
    custom complete                      -> start the growl display; stamp the clock
    custom, enemy escaped past 11 + delta        -> back to attack_hidden, clear escalation
    control_fire, enemy escaped past 7 + delta
        OR the display has lasted longer than the creature's drive-out time
                                         -> back to custom; set escalation if it was the timeout
    anything else                        -> steal
```

**Notes** — the ladder's shape is a hysteresis loop. Each rung has an *arrival* radius and a
*fall-back* radius that is larger, so the creature does not oscillate between rungs at a boundary;
the gap is roughly 4 to 5 world units at every rung. A rebuild that uses one radius per rung
produces packs that flicker.

The escalation flag is what makes the behaviour end. A creature that has been growling for longer
than its section's drive-out time drops back to the approach rung with the escalation flag set,
which shrinks that rung's arrival radius from 6 to **1 world unit** — that is, it walks right up
to the enemy. The pack's message to the player is "leave", and if the player does not leave, the
pack stops asking. The drive-out time is authored per creature section; see
[`../dog/dog.cpp`](../dog/dog.cpp.md).

The growl display is driven through the numbered-animation machine: the composite requests clip 6
(*growl while standing*) by number and clears the request flag itself, then hands control to a
wrapper state. That is why this generic-looking template reaches into fields that only the dog
defines.

## `setup_substates`

**Contract** — called once at the moment a substate becomes active; fills the parameter record the
substate will read for its whole activation. Four substates are parameterised this way: the three
stalking rungs and the retreat.

**Invariants** — the parameter record is copied into the substate by size, so the record type the
composite fills must match the one the substate declares exactly. This is a raw memory copy in the
original; a rebuild with a typed hand-off removes the hazard and loses nothing.

The parameters, beyond the radii already given: all three stalking rungs rebuild their path at the
creature's distance-scaled attack rebuild interval, accelerate aggressively without braking, and
play the idle sound at the section's idle-sound delay — except the growl rung, which plays the
threat sound. The retreat runs 20 units from the enemy's position, with an aggression sound and a
five-second cap.

## `check_behinder`

**Contract** — a two-stage timer answering "has the enemy been behind me long enough that I should
disengage". Stage one starts when the enemy leaves a 120-degree frontal arc and is cancelled if the
enemy re-enters a 90-degree arc. After two seconds in stage one, stage two begins and the answer is
*yes* for three seconds, after which the tracker resets.

```text
FUNCTION check_behinder() -> bool
  IF not yet in stage two
    IF stage one not started
      IF enemy is outside the 120-degree frontal arc
        start stage one
    ELSE
      IF enemy is inside the 90-degree frontal arc   cancel stage one
      IF stage one has lasted less than 2 seconds    RETURN false
      enter stage two; clear stage one
  IF in stage two
    IF stage two has lasted less than 3 seconds      RETURN true
    leave stage two
  RETURN false
```

**Notes** — the two arcs differ deliberately: 120 degrees to *start* suspecting, 90 degrees to
*stop*. That is the same hysteresis idea as the ring radii, applied to an angle, and it stops the
tracker from arming and disarming as the creature turns.

The consequence in play is the "circling" that makes packs hard to melee: a creature that has been
attacked from behind for two seconds retreats 20 units, turns, and re-approaches. The source marks
this as a workaround — the intended fix was a dedicated state that turns in place rather than
retreating — and says so, which makes it a deliberate placeholder rather than a design. A rebuild
is free to write the turn state instead.

## `finalize` / `critical_finalize`

**Contract** — clear the enemy's *aggressive* mark and drop any script-imposed enemy. The forced
exit additionally clears the enemy's *attack started* mark, and does both only when the enemy is
dead.

**Notes** — the asymmetry is a real difference in meaning. A clean exit means the creature is
choosing to stop; a forced exit means something took the creature away, and the marks are cleared
only when the enemy is dead — i.e. when nobody else will need them either.

## `check_home_point`

**Contract** — the standard latch: if we were not going home, ask whether we should start; if we
were, ask whether we have arrived. A commented-out early return beside it would have suppressed
home defence entirely in aggressive home regions; it is marked *intentionally* disabled, so the
shipped behaviour is that even an aggressive pack goes home when pulled out of its territory.

## `remove_links`

**Contract** — when the object being destroyed is the latched enemy, forget it. **Does not call the
base implementation**, so the notice is not forwarded to the eleven substates.

**Notes** — that omission is almost certainly a defect: every other composite in the chapter
forwards the notice down. The substates that latch the enemy themselves therefore keep a dangling
reference through this composite. A rebuild should forward, and should expect the resulting
behaviour to differ from the original in the rare case where an enemy is destroyed mid-attack.
