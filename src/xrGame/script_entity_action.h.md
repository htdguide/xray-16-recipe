# src/xrGame/script_entity_action.h

> One scripted action: eight simultaneous demands on a creature plus the rule that says when they are jointly done.

**Needs** — [`script_movement_action.h`](script_movement_action.h.md) · [`script_watch_action.h`](script_watch_action.h.md) · [`script_animation_action.h`](script_animation_action.h.md) · [`script_sound_action.h`](script_sound_action.h.md) · [`script_particle_action.h`](script_particle_action.h.md) · [`script_object_action.h`](script_object_action.h.md) · [`script_action_condition.h`](script_action_condition.h.md) · [`script_monster_action.h`](script_monster_action.h.md) · [`script_entity_action_inline.h`](script_entity_action_inline.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`base_monster_script.cpp`](ai/monsters/basemonster/base_monster_script.cpp.md) · [`ai_stalker_script_entity.cpp`](ai/stalker/ai_stalker_script_entity.cpp.md) · [`script_entity.cpp`](script_entity.cpp.md) · [`script_entity.h`](script_entity.h.md) · [`script_entity_action.cpp`](script_entity_action.cpp.md) · [`script_entity_action_inline.h`](script_entity_action_inline.h.md) · [`script_entity_action_script.cpp`](script_entity_action_script.cpp.md) · [`script_game_object.cpp`](script_game_object.cpp.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`searchlight.cpp`](searchlight.cpp.md) · [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md)
**Tier floor** — T2: a composite record and a completion predicate

## Purpose

The unit a script queues on an entity. It is deliberately a *fixed aggregate* rather than a
list of demands: all eight channels always exist, each carries its own completion flag, and
an unset channel is simply one that starts already complete. That makes "is this action
finished" a cheap fixed test and means a script can set any subset without the engine
allocating anything.

This header holds the substance even though a `.cpp` exists, because the whole type is
inline; see [`script_entity_action_inline.h`](script_entity_action_inline.h.md) for the
bodies and [`script_entity_action.cpp`](script_entity_action.cpp.md) for the empty
translation unit.

## State

```text
RECORD EntityAction
  movement   : MovementAction      # where to go and how
  watch      : WatchAction         # where to look
  animation  : AnimationAction     # what to play
  sound      : SoundAction         # what to emit
  particle   : ParticleAction      # what to emit visually
  object     : ObjectAction        # what to do with an item
  condition  : ActionCondition     # which channels must finish, and a time limit
  monster    : MonsterAction       # a whole-creature behaviour mode
  user_data  : opaque              # an arbitrary script value carried along, untouched
  started    : bool                # has this action been asked about its completion once
```

**Invariants**

- Every channel record carries a `completed` flag whose *default is true*. An action
  nobody configured is finished the moment it is examined.
- `started` exists so the first completion test always answers "not finished", no matter
  what the channels say. Without it an action whose channels are all trivially complete
  would be retired before it was ever projected onto the entity, and a script's
  "play nothing for two seconds" would take no time at all.

## `set_action(channel)`

**Contract** — one overload per channel, each copying the supplied channel record over the
corresponding field. Plus one taking the opaque user value. There is no partial update: a
channel is replaced whole.

## `completed` — the joint completion rule

**Contract** — answers whether the action as a whole is finished. Never allocates. The
first call on a given action always answers false. The rule has two modes, chosen by
whether the action's condition names any channels at all:

```text
FUNCTION completed() -> bool
  first_time = NOT started
  started = true
  IF first_time THEN RETURN false          # an action always gets one frame

  IF condition names no channels THEN
    # implicit rule: every channel must be done
    RETURN movement.completed AND watch.completed AND animation.completed
       AND sound.completed AND particle.completed AND object.completed
       AND monster.completed
  ELSE
    # explicit rule: only the named channels count, and the time limit is one of them
    remaining = condition.channels
    FOR EACH named channel c IN condition.channels
      IF c is done THEN remove c FROM remaining
    IF condition names a time limit AND the limit has passed THEN
      remove the time limit FROM remaining
    RETURN remaining is empty
```

**Invariants**

- The implicit rule ignores the time limit entirely; a script that wants a timeout must
  name it explicitly.
- The explicit rule's set of names includes the time limit as a peer of the six channels
  and of the monster behaviour, so "finish when the animation ends *or* after two seconds"
  is not expressible — naming both means waiting for both. That is a real limitation of the
  design and shipped scripts work around it by queueing short actions.

## `time_over`

**Contract** — true once the global clock has passed the action's start time plus its
lifetime. Both come from the condition record. The comparison is on a monotonic
millisecond clock.

## `initialize`

**Contract** — resets `started` and forwards to every channel's own reset. Called by the
queue the first time an action reaches the front, and again if it is displaced and
restored. Note the monster channel is *not* reset — it has no per-run state.

## Channel readers

**Contract** — `move`, `look`, `anim`, `particle`, `object`, `cond`, `data` return the
corresponding field for inspection. The script-visible names for these are the *completion
tests*, not the readers (see
[`script_entity_action_script.cpp`](script_entity_action_script.cpp.md)), which is a
frozen confusion: in script, `action:move()` asks whether movement finished.

## `construct(other)`

**Contract** — a copy. The queue copies actions on the way in, so a script may reuse and
mutate the value it passed. The copy is shallow for the opaque user value and for the
particle system handle, which is why the queue, not the action, decides when a particle
system dies.
