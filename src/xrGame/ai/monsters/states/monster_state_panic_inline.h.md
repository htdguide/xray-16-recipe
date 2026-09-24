# src/xrGame/ai/monsters/states/monster_state_panic_inline.h

> Fleeing an enemy it can see: run until fifteen units away and fifteen seconds unseen, pause three
> seconds facing the open ground, repeat — and break out of the pause instantly if the enemy
> reappears.

**Needs** — [`monster_state_panic.h`](monster_state_panic.h.md) · [`state_data.h`](state_data.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`state_look_unprotected_area.h`](state_look_unprotected_area.h.md) · [`monster_state_panic_run.h`](monster_state_panic_run.h.md) · [`monster_state_home_point_attack.h`](monster_state_home_point_attack.h.md)
**Used by** — [`monster_state_panic.h`](monster_state_panic.h.md)
**Tier floor** — T3: a two-way alternation plus a pre-emption rule

## Purpose

The behaviour that makes weak creatures weak. It is selected instead of combat when the brain's
weighting says this animal should not fight this enemy — because the species is timid, because it
is badly wounded, or because it is alone. What the player sees is an animal that runs, stops,
looks back at them, and runs again.

The stop is the whole point. A creature that only ran would vanish; the pause is where the player
catches up, and where the creature decides whether it is still being chased.

## State

`Stateless.`

## `reselect_state`

**Contract** — the fall-back-into-territory leaf wins whenever its own start condition holds,
re-tested on every selection. Otherwise alternate: bolt, then pause, then bolt again.

```text
FUNCTION next_leaf(previous) -> leaf
  IF fall_back_home.can_start()  RETURN fall_back_home   # re-tested every time
  IF previous == bolt            RETURN pause_facing_open_ground
  RETURN bolt
```

**Notes** — the territorial branch reuses the **combat** fall-back leaf
([`monster_state_home_point_attack.h`](monster_state_home_point_attack.h.md)), not the danger one.
That is the right choice and a subtle one: the danger retreat runs to a single covered spot and
settles, which is fatal with a pursuer; the combat version hops from claimed spot to claimed spot
continuously, which keeps a fleeing creature moving while still drawing it homeward. A rebuild that
"unifies" the two go-home leaves loses the distinction.

Note also that the fall-back leaf's start condition includes the enemy-unreachable clause, so a
panicking creature that has put a wall between itself and its enemy is also pulled home.

## `check_force_state` — the pre-emption

**Contract** — while the pause leaf is running, cut straight back to bolting if the enemy was seen
this very frame, or if a hit has landed within the last five seconds.

```text
FUNCTION check_force_state()
  IF current_leaf != pause_facing_open_ground  RETURN
  IF enemy_memory.time_last_seen == now()              select(bolt); RETURN
  IF hit_memory.last_hit_time + 5000 > now()           select(bolt)
```

**Notes** — this is the mechanism that makes the pause feel like an animal checking whether it is
safe rather than a scripted beat. The base machinery only re-selects a leaf when the current one
*completes*; forcing is the escape hatch that lets a leaf be abandoned mid-way, and this is the one
place in this behaviour that uses it.

*Seen this frame*, not seen recently. The visibility test compares the last-seen timestamp against
the current frame's time, so it is true only while the enemy is actually in view right now. A
creature that pauses behind cover stays paused even though it saw the enemy a second ago; step into
view and it bolts on that frame.

*Five seconds* for the hit is the opposite kind of test — a window, not an instant — because a
creature that is being shot at should keep running even between hits. The number is hard-coded.

The pre-emption never fires during the bolt leaf, which is correct: there is nothing to escalate
to.

## `setup_substates`

**Contract** — fill the pause leaf's parameters. The bolt and fall-back leaves configure
themselves.

```text
pause_facing_open_ground:
  action    = stand idle, scared posture
  time_out  = 3000 ms
  voice     = panic, delay = section key "attack_sound_delay"
```

**Notes** — the three-second pause is hard-coded and is the tempo of the whole behaviour: run,
three seconds, run. The panic voice is what the player hears; `attack_sound_delay` throttles its
repetition and is the only authored number here.

The leaf turns the creature toward the **least covered** direction, not toward the enemy — the same
rule the fear behaviours use. A fleeing animal watches the ground it would be charged across. Since
that direction is recomputed when the leaf is entered, and the creature has just run, it is
typically roughly back the way it came, which is why the pause reads as looking back.

The entry step exists and does nothing but call the base. It is not a placeholder for missing work:
this behaviour genuinely has no per-entry state.
