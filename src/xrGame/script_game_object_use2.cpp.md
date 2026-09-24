# src/xrGame/script_game_object_use2.cpp

> The per-species monster controls: the levers a script pulls on a bloodsucker, a burer, a poltergeist or a zombie that no other creature has, plus the two perception snapshots every monster can hand back.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`script_sound_info.h`](script_sound_info.h.md) · [`script_monster_hit_info.h`](script_monster_hit_info.h.md) · [`ai/monsters/bloodsucker/bloodsucker.h`](ai/monsters/bloodsucker/bloodsucker.h.md) · [`ai/monsters/poltergeist/poltergeist.h`](ai/monsters/poltergeist/poltergeist.h.md) · [`ai/monsters/burer/burer.h`](ai/monsters/burer/burer.h.md) · [`ai/monsters/zombie/zombie.h`](ai/monsters/zombie/zombie.h.md) · [`ai/monsters/monster_home.h`](ai/monsters/monster_home.h.md) · [`ai/monsters/control_animation_base.h`](ai/monsters/control_animation_base.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation only

## Purpose

One of the nine files the [game object facade](script_game_object.h.md) is split across,
and the most openly ad hoc. The other eight export capabilities that many entity kinds
share; this one exports **one species' ability at a time**, because the set-piece
encounters the games ship need a script to override exactly one creature's signature
behaviour and nothing else.

Every method follows the
[guarded-delegation pattern](script_game_object.cpp.md#the-guarded-delegation-pattern).
Two escalation choices recur and they are not uniform — see the notes.

## State

`Stateless.` Two methods construct and return a record by value; see below.

## Bloodsucker: visibility

**Contract** — the cloaking creature's defining ability, exposed as four separate levers
that a rebuild must keep distinct because they act at different levels:

- `force_visibility_state(state)` and `get_visibility_state` — pin the creature to a
  discrete cloak state, or read the current one. The read's fallback on a non-bloodsucker
  is **fully visible**, which is the safe answer: a script that mistakes its subject gets
  "you can see it", never "it is invisible".
- `set_invisible(on)` — trigger the creature's own cloak activation or deactivation,
  animation and effects included. This is the *behaviour*, not the state.
- `set_manual_invisibility(on)` — hand cloak control to the script and take it away from the
  creature's brain, or give it back.
- `set_vis_state(value)` — a third spelling that snaps the creature visible on `1` and
  invisible on `-1`, and **does nothing at any other value**, silently. The comparison is
  against an exact real number, which is a trap: a script computing the argument rather
  than writing the literal gets no effect and no message.

**Invariants** — `set_manual_invisibility` must be turned on before `set_invisible` sticks,
or the creature's brain reasserts its own choice on the next update. Nothing enforces the
ordering and nothing reports it; it is the most common script mistake against this
creature.

## Bloodsucker: the drag attack

**Contract** — `bloodsucker_drag_jump(victim, animation, position, factor)` starts the
creature's scripted grab: leap to the position, play the named animation, and drag the
named victim. `set_enemy(victim)` forces the target directly, bypassing the creature's own
enemy selection. `off_collision(on)` disables the creature's collision so a scripted leap
can pass through geometry it would otherwise catch on. `set_alien_control(on)` hands the
whole creature to an external controller.

**Invariants** — the victim is converted to a living entity and the conversion is **not
checked**: passing a crate hands the creature nothing to drag. A rebuild should report it.

## Burer, poltergeist, zombie

**Contract** — one pair or one pair-and-a-half each:

- burer: `set_force_gravi_attack(on)` / `get_force_gravi_attack` — force the gravity throw
  regardless of what the creature's own brain would choose.
- poltergeist: `set_actor_ignore(on)` / `get_actor_ignore` — make it disregard the player,
  used for the encounters where it must haunt a place rather than hunt.
- zombie: `fake_death_fall_down` → answers whether it actually went down, and
  `fake_death_stand_up`. The fall can be refused — a zombie already down, or mid-animation,
  will not go down again — which is why one half of the pair returns an answer and the
  other does not.

## Any monster

**Contract** —

- `set_force_anti_aim(on)` / `get_force_anti_aim` — force the creature's aim-disrupting
  behaviour on.
- `set_override_animation(name)` / `clear_override_animation` — pin one animation over
  whatever the creature's own animation control would play, and release it.
- `force_stand_sleep_animation(index)` / `release_stand_sleep_animation` — the same idea for
  the sleeping pose, selected by index into the creature's authored set rather than by name.
- `skip_transfer_enemy(on)` — stop this creature passing its enemy on to its pack.
- `berserk()` — make it attack anything, its own kind included.
- `set_custom_panic_threshold(value)` / `set_default_panic_threshold` — retune or restore
  the health fraction below which the creature flees.
- `set_home(name | vertex, min_radius, max_radius, aggressive, mid_radius)` and
  `remove_home` — bind the creature to a place: it stays within the outer radius, defends
  the inner one, and the aggressive flag decides whether it attacks intruders or merely
  withdraws. The middle radius is a third band between them.

**Invariants** — the home is named either by a level location name or by a navigation
vertex, and the two forms are otherwise identical. The vertex form is what the engine uses
when restoring a save, since a name may no longer resolve.

## `sound_info` and `monster_hit_info`

**Contract** — return the creature's last remembered sound event and last recorded hit as
value records, or an empty record when nothing is remembered. A non-monster produces an
empty record *and* a script error.

```text
FUNCTION sound_info -> SoundInfo
  result = empty record
  IF this is not a monster THEN log script error; RETURN result
  IF the creature remembers no sound THEN RETURN result
  (event, dangerous) = the creature's remembered sound
  emitter = event.source's facade, but only IF the source still exists
  result.set(emitter, dangerous, event.position, event.power, event.time)
  RETURN result
```

**Invariants**

- The remembered emitter is checked for destruction before its facade is taken, and
  replaced with nothing when it has gone. A creature's memory outlives the things it
  remembers — that is what memory is for — so every reader of it must tolerate a dead
  source. The same check appears in the hit-info reader. A rebuild that skips it hands
  scripts a dangling handle whenever the shooter dies first.
- Both return **by value**. The script gets a snapshot; mutating it changes nothing, and
  the creature may overwrite its own memory the next frame.
- An empty record and a record for a source that has since died are indistinguishable to
  the script. That is a real limitation of the exported shape and is worth knowing before
  reproducing it.

**Notes**

Escalation is inconsistent across this file and the inconsistency is visible in gameplay.
The bloodsucker, burer, poltergeist and zombie methods **log** on a failed downcast; the
generic monster methods — `skip_transfer_enemy`, `set_home`, `remove_home`, `berserk`, the
panic thresholds — **do nothing silently**. So a script that calls `berserk` on a crate
gets no diagnostic at all. A rebuild should log everywhere; nothing depends on the silence.

Several error messages name the wrong method — the zombie stand-up reports the fall-down,
two bloodsucker methods report each other's names. Cosmetic, but it means the log cannot be
trusted to identify the call site.
