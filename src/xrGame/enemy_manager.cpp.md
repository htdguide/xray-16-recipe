# src/xrGame/enemy_manager.cpp

> Which of the creatures I know about am I fighting right now: the filter that says who counts as an enemy, the score that ranks them, and the hysteresis that stops a creature flicking between two targets.

**Needs** — [`enemy_manager.h`](enemy_manager.h.md) · [`enemy_manager_inline.h`](enemy_manager_inline.h.md) · [`object_manager.h`](object_manager.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`ef_pattern.h`](ef_pattern.h.md) · [`autosave_manager.h`](autosave_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrEngine/profiler.h`](../xrEngine/profiler.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a scored selection over a short list, run per creature per update; the cost is in the world-state evaluation it calls into

## Purpose

Every creature's memory holds a list of other creatures it has seen or been hit by. This
file reduces that list to **one** enemy — the target the planner will aim at, take cover
from and chase — and it is the only place in the AI that makes that choice.

Three things make it more than a maximum-of-a-list. First, the *useful* filter: not every
hostile creature is worth fighting, and a stalker is allowed to walk past a weak mutant.
Second, the *score*, which is not distance but a blend of distance, recent damage, current
visibility and the world-state evaluator's estimate of who would win. Third, and most
visible in play, the **inertia**: a creature that re-picks its target every update would
swing its weapon between two equally scored enemies several times a second, so a change of
enemy has to earn its way past a timeout that depends on who is being dropped and who is
being taken up.

It also participates in autosave gating, which is unrelated to target selection and
explained below.

## State

```text
RECORD EnemyManager EXTENDS ObjectManager<EntityAlive>
  object                   : CustomMonster      # the creature this belongs to
  stalker                  : optional<Stalker>  # the same creature, if it is a stalker
  ignore_monster_threshold : real               # score above which a human may ignore a monster
  max_ignore_distance      : real               # ...but only from at least this far away
  ready_to_save            : bool               # see the autosave section
  last_enemy_time          : int (ms)           # global clock at the last update that had an enemy
  last_enemy               : optional<EntityAlive>   # the last enemy there was, kept after it is gone
  last_enemy_change        : int (ms)           # global clock at the last accepted change of enemy
  enable_enemy_change      : bool               # false pins the current enemy against loss of sight
  smart_cover_enemy        : optional<EntityAlive>   # a forced enemy that overrides selection entirely
  useful_callback          : optional<script callback>  # a script veto on any candidate
```

Invariants:

- **`smart_cover_enemy`, while set and alive, *is* the selection.** The inherited selection
  is computed anyway but not consulted. This is how a creature occupying a smart cover is
  told which way to point, overriding its own judgement.
- **`last_enemy` outlives the selection.** It is what "I was fighting someone a moment ago"
  is asked against, together with `last_enemy_time`; it is cleared only when the entity it
  names is destroyed.
- **Every reference held here is cleared when the referent is destroyed** — see
  `remove_links`. Enemy references are raw and the entity may vanish at any update.

## `useful`

**Contract** — the filter: may this creature be considered as an enemy at all. Called for
every candidate on every selection pass. Yields false for the dead, for entities the AI
cannot perceive, for oneself, for non-enemies by faction relation, for entities standing
nowhere the navigation mesh recognises, and for monsters a human may ignore. Finally defers
to the script veto if one is installed.

```text
FUNCTION useful(candidate) -> bool
  IF NOT candidate.alive:                            RETURN false
  IF candidate is not flagged visible-to-AI:         RETURN false
  IF candidate is self OR relation is not hostile:   RETURN false
  IF no level graph OR candidate's navigation vertex is invalid: RETURN false

  # the "walk past the weak mutant" rule — all four must hold
  IF self is human AND candidate is not human
     AND NOT expedient(candidate)
     AND evaluate(candidate) >= ignore_monster_threshold
     AND distance(self, candidate) >= max_ignore_distance
    RETURN false

  IF a script veto is installed: RETURN veto(self, candidate)
  RETURN true
```

**Invariants** — the navigation-vertex test is what keeps a creature from picking a fight
with something it can never reach: everything downstream (pathing, cover, position
evaluation) is expressed in navigation vertices, so an enemy off the mesh would produce
plans that cannot be executed.

**Notes** — the ignore rule reads oddly and is worth unpacking. `evaluate` returns a
*penalty* — lower is a better target — so a score **at or above** the threshold means a
*bad* target. A human ignores a monster when it is a bad target, is far away, and has not
done anything to make fighting it expedient. The default threshold of 1 with a default
ignore distance of 0 means: by default, ignore any monster whose score is at least 1 at any
distance — and since real scores start around ten thousand, the default is effectively
"humans ignore monsters", which each creature's configuration then overrides.

The visible-to-AI flag is a spatial-index property, not a line-of-sight test. It marks
entities that participate in AI perception at all; the actual seeing is the visual memory's
job, and its result only *weights* the score, it does not gate candidacy. That is deliberate
— a creature keeps fighting an enemy that ducks behind a wall.

## `evaluate` / `do_evaluate`

**Contract** — the score. Lower is a more attractive target. Not a pure function: it writes
the creature pair into the shared world-state evaluator and reads a victory-probability
estimate back, and it has one genuine side effect on the autosave gate. Called once per
candidate per selection pass.

```text
FUNCTION evaluate(candidate) -> real
  IF candidate is the player: ready_to_save = false        # see the autosave section

  IF candidate is a wounded stalker
    IF this creature's squad has already assigned someone to finish him
      RETURN 0                                             # top priority, see Notes
    RETURN squared_distance(self, candidate)               # otherwise: nearest wounded

  penalty = 10000
  IF candidate is who last hit me
    penalty = penalty - (candidate is the player ? 1500 : 500)
  IF I can see candidate now
    penalty = penalty - 1000

  world_state.member = self ; world_state.enemy = candidate
  RETURN penalty
       + squared_distance(self, candidate) / 100
       + world_state.victory_probability / 100
```

**Notes** — the constants are the tuning of the whole combat AI and each one is a ratio
against the distance term, which is *squared* distance divided by a hundred. At ten metres
squared distance contributes one point; at a hundred metres, a hundred points. So the
visibility bonus of a thousand dominates distance out to well past any engagement range: a
creature you can see now is preferred over one you cannot, almost regardless of range. Being
hit is worth half that, and being hit *by the player* three times that — the player is
weighted to stay the centre of attention, which is a design decision about a single-player
game, not an emergent one.

The wounded branch is a different scale entirely: zero or raw squared distance, both far
below the ten-thousand base, so **any** wounded stalker outranks every healthy enemy. That
is the "finish him off" behaviour, and the squad-level assignment check gives an absolute
zero to the one the squad has already nominated so that two creatures do not both converge
on the same body.

The victory-probability term comes from the world-state evaluation machinery and is divided
by a hundred like the distance, so it contributes at most a few points — it breaks ties, it
does not decide. The conditional compilation around it (`USE_EVALUATOR`) has a disabled
alternative that scores purely on visibility and distance; it is dead and the evaluator path
always compiles.

## `expedient`

**Contract** — private-ish (protected): is fighting this candidate worth it, independent of
the score. True when the world-state evaluator's expediency pattern says so, or when this
candidate has hit me. Writes into the shared evaluator.

**Notes** — the "it hit me" clause is what makes the ignore rule non-absolute: a human may
walk past a weak monster, but once the monster bites him he fights back. Without it, a
creature configured to ignore monsters would be permanently unable to defend itself.

## `update`

**Contract** — the per-update entry point. Manages the autosave gate around the selection,
runs the selection, and records the enemy and the time if one was chosen. Called once per
creature per AI update.

```text
FUNCTION update()
  IF NOT ready_to_save: autosave.dec_not_ready()    # release last update's hold
  ready_to_save = true
  try_change_enemy()                                # evaluate() may clear ready_to_save
  IF selected exists
    last_enemy_time = global_clock
    last_enemy      = selected
  IF NOT ready_to_save: autosave.inc_not_ready()    # take a fresh hold
```

**Notes** — the autosave gate is the non-obvious half of this file. A single-player autosave
must not happen while the player is being evaluated as someone's enemy, because the saved
state would capture a creature mid-decision about the player. So each update: release any
hold taken last time, optimistically assume no hold is needed, run the selection (which
clears the flag if the player appeared as a candidate), and take a hold if it did. The
counter is engine-wide; many creatures hold it at once and the save waits for all of them.

The release-then-reacquire shape means the counter can dip to zero between the two lines of
one creature's update, which is safe only because nothing can save in the middle of an AI
update. A rebuild with a different save trigger needs the *decision* — the player being
someone's live enemy candidate blocks autosave — not this counter dance.

`set_ready_to_save` is the same release, callable from outside for a creature that is being
taken out of the world mid-hold.

## `try_change_enemy`

**Contract** — private. Decides whether to re-run selection at all, re-runs it, and applies
the inertia rule to the result. Notifies the creature when the enemy actually changed.

```text
FUNCTION try_change_enemy()
  previous = selected
  process_wounded() -> only_wounded
  IF NOT need_update(only_wounded): RETURN

  base.update()                       # the scored selection over the filtered candidates

  IF selected != previous
    IF selected exists AND previous exists
      on_enemy_change(previous)       # may put `previous` back
    ELSE
      last_enemy_change = global_clock
  IF selected != previous             # re-tested: on_enemy_change may have reverted it
    object.on_enemy_change(previous)
```

**Invariants** — the second comparison is deliberately a re-test, not a cached result. The
inertia rule inside `on_enemy_change` can restore the previous enemy, and in that case the
creature must **not** be notified of a change that did not happen — a notification would
restart aiming, re-plan, and produce exactly the twitch the inertia exists to prevent.

## `process_wounded` / `remove_wounded`

**Contract** — private. Determines whether every remaining candidate is a wounded stalker.
If so, nothing is removed and the caller is told. If not, every wounded stalker is
**removed from the candidate list** for this pass.

**Notes** — this is a priority rule expressed as a filter, and it is stronger than the score
could express. Healthy enemies always crowd out wounded ones; wounded ones become candidates
only when there is nothing else left. The score's wounded branch — which ranks wounded
enemies above everything — therefore only ever runs when the list is all wounded. The two
mechanisms together read as: *never turn away from a live threat to finish a wounded man,
but when the fight is over, finish him.*

The removal is destructive to the candidate list, which is rebuilt each pass from memory, so
it costs nothing beyond this pass.

## `need_update`

**Contract** — private. Should selection re-run this pass? Yields true when there is no
enemy, when the enemy is dead, when relations changed so the enemy is no longer hostile,
when the enemy is out of sight and changing enemies is permitted, when the current enemy is
a wounded stalker and non-wounded candidates exist, or when something has hit this creature
since the last enemy change and the hitter is a live hostile. Otherwise false — keep the
current enemy and skip the whole scored pass.

```text
FUNCTION need_update(only_wounded) -> bool
  IF no selection:                             RETURN true
  IF selection is dead:                        RETURN true
  IF selection is no longer hostile:           RETURN true
  IF enemy_change_enabled AND selection not visible now: RETURN true
  IF only_wounded:                             RETURN false   # nothing better can exist
  IF selection is a wounded stalker:           RETURN true    # something better may exist
  IF I was hit since the last enemy change
     AND the hitter is alive and hostile:      RETURN true
  RETURN false
```

**Notes** — this is the cheap gate in front of an expensive pass, and each clause is a
*reason the answer might have changed*. The hit clause is the interesting one: being shot
from an unexpected direction must be able to reopen the decision even while the current
enemy is perfectly visible, because otherwise a creature engaged with one target is deaf to
a second one flanking it. The comparison is against the last enemy *change*, not the last
update, so one hit reopens the question exactly once.

The visibility clause is gated on `enable_enemy_change`, the external pin. A creature told
to hold its target keeps it through loss of sight — used when a script or a smart cover
wants a creature committed.

## `on_enemy_change`

**Contract** — protected. Called when selection produced a different enemy while both the
old and the new exist. Either accepts the change and stamps the time, or **reverts the
selection to the previous enemy**. Four rules, in order.

```text
FUNCTION on_enemy_change(previous)
  IF previous is dead:                          accept        # nothing to be loyal to
  IF previous was wounded and the new one is not: accept       # a live threat trumps a body
  IF enemy_inertia(previous):                   selected = previous ; RETURN   # too soon
  IF previous not visible AND new one visible:  accept
  accept
```

**Invariants** — the order matters: the two escape hatches come *before* the timeout, so a
dead or wounded previous enemy is dropped immediately regardless of how recently the enemy
changed. Without that, a creature would stand aiming at a corpse for the full inertia
period.

**Notes** — the last two branches are identical in effect, both merely stamping the time;
the visibility comparison that separates them changes nothing and is vestigial. What matters
is that once the timeout has passed, the change is accepted unconditionally.

## `enemy_inertia`

**Contract** — private. Is it too soon to change enemy? Three different timeouts, chosen by
who is involved.

```text
FUNCTION enemy_inertia(previous) -> bool
  IF the NEW enemy is the player:      RETURN global_clock <= last_change + 0 ms
  IF the PREVIOUS enemy is the player: RETURN global_clock <= last_change + 6000 ms
  RETURN global_clock <= last_change + 3000 ms
```

**Notes** — these three numbers are the feel of enemy switching and they are deliberately
asymmetric around the player.

*Switching **to** the player is instant.* Zero timeout: the moment the player outranks the
current target, the creature turns. The player must never watch an enemy finish a three
-second commitment to a mutant before noticing him.

*Switching **away from** the player costs six seconds.* Double the general timeout. Once a
creature is fighting the player it stays fighting the player, so the player is not casually
dropped for a better-scoring target.

*Everything else costs three seconds.* Long enough to stop oscillation between two
similar-scoring enemies, short enough to react to a real change.

Written against the global wall clock, so the timeouts are real time and do not scale with
the AI update rate — a distant creature updated rarely still honours the same three seconds.

## `change_from_wounded`

**Contract** — private. True only in the one specific transition: the previous enemy was a
wounded stalker and the new one is a healthy stalker. Used solely to bypass inertia.

**Notes** — it tests both ends for being stalkers, so the escape does not fire when turning
from a wounded stalker to a monster. That looks like an omission rather than a decision; the
"wounded" state is a stalker-only concept in this engine, so the previous-end test is
necessary, but the new-end test excludes a legitimate case.

## `remove_links`

**Contract** — called when any game object is destroyed. Removes it from the candidate list
and clears both the last-enemy reference and the current selection if either names it. Must
be called for every destruction, because every reference here is raw.

**Notes** — the candidate search compares references rather than identifiers, and the source
notes this is safe because no field of the entity is touched during the search. That is a
C++ lifetime argument; a rebuild whose references are safe to compare after death needs no
such note, but still needs the *clearing*, because a destroyed enemy must not remain
selected.

## `reload`

**Contract** — reads the two ignore-rule tunables from the creature's configuration section,
defaulting to a threshold of 1 and an ignore distance of 0, and clears the selection history
and the script veto. Called when the creature's section changes.

**Notes** — the paired restore operations (`restore_ignore_monster_threshold` and its
distance sibling) re-read the same two values from the creature's own section, so a script
that temporarily raises the ignore threshold can put it back without knowing the original
value. That is the whole reason those two exist as separate calls from the setters.
