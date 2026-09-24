# src/xrGame/ai/monsters/burer/burer_state_attack_inline.h

> The burer fight, arbitrated in a fixed order every tick: gravity if it is off cooldown, shield if it has just been hurt, anti-aim if the player is drawing a bead, telekinesis if there is anything to throw — and otherwise keep its distance and face the player.

**Needs** — [`burer_state_attack.h`](burer_state_attack.h.md) · [`burer_state_attack_gravi.h`](burer_state_attack_gravi.h.md) · [`burer_state_attack_tele.h`](burer_state_attack_tele.h.md) · [`burer_state_attack_shield.h`](burer_state_attack_shield.h.md) · [`burer_state_attack_antiaim.h`](burer_state_attack_antiaim.h.md) · [`burer_state_attack_run_around.h`](burer_state_attack_run_around.h.md) · [`burer_state_attack_melee.h`](burer_state_attack_melee.h.md) · [`state_look_point.h`](../states/state_look_point.h.md) · [`state_move_to_restrictor.h`](../states/state_move_to_restrictor.h.md) · [`monster_state_attack_run.h`](../states/monster_state_attack_run.h.md) · [`ai_monster_squad.h`](../ai_monster_squad.h.md) · [`burer.h`](burer.h.md)
**Used by** — [`burer_state_attack.h`](burer_state_attack.h.md)
**Tier floor** — T2: arbitration between abilities and movement, per tick

## Purpose

The burer has five things it can do to an enemy and this file decides, every tick, which one. Two properties make the arbitration non-trivial and both are load-bearing.

First, the burer's attacks are not interruptible: once one is chosen it runs to its own completion before anything else is considered. Second, the priority order is fixed in code — it is not a weighted choice and not data — so a rebuild that reorders the tests produces a visibly different creature.

## State

See [`burer_state_attack.h`](burer_state_attack.h.md). Invariants:

- `waiting_for_substate_end` means a committed sub-attack owns the tick; while it is set, arbitration is skipped entirely.
- `lost_health_since_check` is a one-shot edge: it is raised when health drops by more than the threshold and cleared by whichever consumer reads it first, so one wound triggers at most one reaction.
- `allow_anti_aim` is the raise/ask/lower latch, exactly as in [`chimera_attack_state_inline.h`](../chimera/chimera_attack_state_inline.h.md).

```text
health_delta        = 0.01          # fraction of health that counts as "hurt"
aim_tolerance       = 20 degrees
runaway_hold        = 5000 ms       # spacing between run-arounds when in close
```

Read from the creature every tick: `runaway_distance`, `normal_distance` — see [`burer.cpp`](burer.cpp.md).

## Substates registered

```text
gravi, tele, shield, anti_aim        the four attacks
run_around                           break away and reposition
face_enemy   -> shared "look at a point" state, used as an idle
attack_run   -> shared "run at the enemy" state
move_to_restrictor -> shared state that walks the creature back inside its
                      permitted volume when it has strayed out
melee        -> registered and never selected; see burer_state_attack_melee_inline.h.md
```

## `initialize`

**Contract** — Samples the creature's current health as the baseline for the wound detector, clears every flag and the run-around cooldown.

## `execute` — the arbitration

**Contract** — One decision and one substate tick. Never blocks.

```text
FUNCTION execute()
  tell the squad this creature is attacking this enemy
  clear any override animation left over from a finished sub-attack

  IF health <= last_health - health_delta
     last_health = health ; lost_health_since_check = true

  # 1. A committed sub-attack owns the tick.
  IF waiting_for_substate_end
     IF NOT current.check_completion()
        current.execute() ; RETURN
     waiting_for_substate_end = false
     select(face_enemy)          # a deliberate no-op state, see Notes

  # 2. Poll the four attacks, in this order.
  allow_anti_aim = true
  anti_aim_ready = anti_aim.check_start_conditions()
  allow_anti_aim = false
  gravi_ready    = gravi.check_start_conditions()
  shield_ready   = shield.check_start_conditions()
  tele_ready     = tele.check_start_conditions()

  IF gravi_ready                                   chosen = gravi
  ELSE IF lost_health_since_check AND shield_ready chosen = shield ; clear the flag
  ELSE IF anti_aim_ready                           chosen = anti_aim
  ELSE IF tele_ready AND current != run_around     chosen = tele
  ELSE                                             chosen = none

  IF chosen != none
     select(chosen) ; current.execute()
     waiting_for_substate_end = true ; RETURN

  # 3. No attack available: position.
  distance = |enemy_position - own_position|
  too_close = distance < runaway_distance
  in_range  = distance < normal_distance

  IF current == move_to_restrictor AND NOT current.check_completion()
     current.execute() ; RETURN
  IF move_to_restrictor.check_start_conditions()
     select(move_to_restrictor) ; current.execute() ; RETURN

  IF current == run_around
     IF current.check_completion()
        IF too_close  next_runaway_allowed_tick = now() + runaway_hold
     ELSE
        current.execute() ; RETURN

  IF lost_health_since_check OR (too_close AND now() > next_runaway_allowed_tick)
     clear the flag ; select(run_around)
  ELSE IF NOT in_range
     select(attack_run)
  ELSE
     select(face_enemy)
     IF not aimed within aim_tolerance
        force the left or right turn clip, whichever side the enemy is on
        turn toward the enemy
     action = stand_idle
     RETURN                      # note: face_enemy's own tick is skipped

  current.execute()
```

**Notes** — Several decisions here are invisible in the code's shape and visible in play.

*Gravity outranks everything*, including the shield, so a burer that is being shot will still open with a wave if the wave is off cooldown. The shield is second and is gated on having just been hurt, which makes it read as a reaction rather than a tactic. Anti-aim is third: it fires when the player is lining up a shot, which the shared anti-aim ability detects.

*Telekinesis is suppressed immediately after a run-around*. The test explicitly refuses to start telekinesis when the current substate is the run-around, because the creature has just moved and the objects it surveyed are no longer where it thought.

*The face-enemy state is used as a deliberate no-op.* After a committed sub-attack finishes, the tree selects it rather than falling through to a new decision. The reason is recorded in the source: with a short enough cooldown a sub-attack could be re-selected on the very tick it completed and never release the creature. Inserting one idle tick breaks that.

*The idle branch never runs its own state's tick* — it returns early after issuing the turn and the stand action. The selected state exists so the debugging display and the "what is this creature doing" query have an answer, not to do work.

*Restrictor handling outranks all positioning.* A burer that has strayed outside its permitted volume walks back in before it will run around or close distance, and that walk is not interruptible by the positioning logic.

## `check_control_start_conditions` — the arbitration hook

**Contract** — Allows the anti-aim ability to start only while `allow_anti_aim` is raised; permits every other ability unconditionally.

**Notes** — Without this, the shared anti-aim ability would fire whenever it decided the player was aiming, regardless of whether the attack tree had chosen it. The latch makes the ability's opinion advisory and the tree's decision authoritative. The same pattern, for the same reason, appears in the chimera's attack.

## `finalize` / `critical_finalize`

**Contract** — Both clear any override animation before delegating. That is the whole teardown: the sub-attacks own their own cleanup, and the tree owns only the forced clip it may have set in the idle branch.
