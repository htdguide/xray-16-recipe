# src/xrGame/script_entity.cpp

> The action queue that lets a script drive a creature directly, bypassing its brain.

**Needs** — [`script_entity.h`](script_entity.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`script_entity_space.h`](script_entity_space.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`patrol_path_manager.h`](patrol_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`script_entity.h`](script_entity.h.md)
**Tier floor** — T2: sequencing and dispatch; the bone transform maths is the only numeric part

## Purpose

An entity normally decides for itself what to do, through its brain and planner. A script
can instead take it *under script control*: from then on the entity's behaviour is a queue
of authored actions, each a bundle of up to eight simultaneous demands — walk here, look
there, play this animation, emit this sound, emit these particles, use this item, behave
like this, and a completion condition tying them together. This file owns that queue and
the translation from one queued action into commands on the entity's movement, animation,
sound and particle machinery.

It is a mixin, not a class of its own: concrete entity types inherit it alongside their
normal behaviour, which is why its hooks are named after the entity lifecycle and are
expected to be chained from the owner's versions (see
[`script_object.cpp`](script_object.cpp.md) for the pattern).

## State

```text
RECORD ScriptEntity
  object            : GameObject       # the entity this mixin is half of; set once at construction
  monster           : optional<Monster># the same entity, if it has movement and memory;
                                       # none for objects that merely animate
  initialized       : bool             # true between net_spawn and net_destroy
  can_capture       : bool             # may a script take control at all
  under_script      : bool             # is a script currently in control
  controlling_script: text             # which one — checked on release, so two scripts
                                       # cannot steal control from each other
  queue             : queue<EntityAction>   # the pending actions, front is current
  current_action    : optional<EntityAction># identity of the action last initialized,
                                       # used to detect "the front changed"
  script_animation  : optional<Motion> # the motion actually playing
  next_animation    : optional<Motion> # the motion resolved from the current action
  use_anim_mover    : bool             # let the animation drive the entity's position
  current_sound     : optional<Sound>  # the one sound the queue may have playing
  saved_sounds      : list<SavedSound> # heard-sound events deferred to a safe point

RECORD SavedSound
  emitter_id  : int (16-bit)
  sound_type  : int
  position    : vector
  power       : real
```

**Invariants**

- At most one sound and one particle system are alive per action, both owned by the front
  action and destroyed when it finishes.
- `current_action` is compared by identity to the queue's front to decide whether to
  re-initialize. Re-initializing an action that is already running would reset its
  completion flags and hang the queue.
- Taking control installs a per-frame animation callback on the entity's visual and
  releasing it removes that callback. The two must balance or the callback outlives the
  control.

## `set_script_control(enabled, script_name)`

**Contract** — takes the entity under script control or releases it. Fails, with a script
error and no state change, unless the request actually *changes* the state and, on release,
the releasing script's name matches the one that took control. A take also fails silently
when the entity has been marked as not capturable. Taking control installs the visual
callback; releasing it removes the callback and clears all script data, which discards the
queue.

**Invariants** — the guard is the whole point: a script may not release an entity it did
not capture, and may not capture one twice. Without it two scripts fight over one creature
and the loser's actions leak.

```text
FUNCTION set_script_control(enabled, name)
  IF NOT (enabled XOR under_script) THEN log error; RETURN     # no actual transition
  IF NOT enabled AND name != controlling_script THEN log error; RETURN
  IF enabled AND NOT can_capture THEN RETURN
  IF enabled THEN add_visual_callback(on_animation_frame)
  ELSE            remove_visual_callback(on_animation_frame)
  under_script       = enabled
  controlling_script = name
  IF NOT enabled THEN reset_script_data()
```

## `add_action(action, high_priority)`

**Contract** — copies an action onto the queue. Without priority it goes to the back.
With priority it goes to the *front*, and the action it displaces is finished and
re-created fresh, so that the displaced action restarts cleanly rather than resuming
half-played. Adding to an empty queue on a spawned entity starts processing immediately;
otherwise processing waits for the next update.

```text
FUNCTION add_action(action, high_priority)
  was_empty = queue is empty
  IF high_priority AND NOT was_empty THEN
    replacement = copy of queue.front          # a pristine copy of the displaced action
    finish_action(queue.front)                 # release its sound and particles
    queue.front = replacement
    queue.push_front(copy of action)
  ELSE
    queue.push_back(copy of action)
  IF was_empty AND initialized THEN process()
```

## `process` — the queue pump

**Contract** — runs on every scheduled update while under script control, and again
whenever an animation completes. It retires every finished action at the front, then
projects the new front onto the entity. Allocates nothing beyond the action copies.
The ordering below is the load-bearing part of the file.

```text
FUNCTION process()
  WHILE queue is not empty
    action = queue.front
    IF action is not current_action THEN action.initialize()   # first sight of it
    current_action = action
    IF NOT action.completed() THEN BREAK
    finish_action(action)                                      # free sound + particles
    fire callback(action_removed)
    queue.pop_front()
  IF queue is empty THEN RETURN

  # Assign the front action's six channels. For each, note whether assignment
  # *newly* completed the channel and, if so, tell the script.
  assign_watch(action)      ; IF newly completed THEN fire callback(action_watch)
  assign_animation(action)
  assign_sound(action)      ; IF newly completed THEN fire callback(action_sound)
  assign_particles(action)  ; IF newly completed THEN fire callback(action_particle)
  assign_object(action)     ; IF newly completed THEN fire callback(action_object)
  assign_movement(action)   ; IF newly completed THEN fire callback(action_movement)
  IF animation channel not completed THEN start_script_animation()
  assign_monster_action(action)
```

**Invariants**

- Animation is assigned *before* movement and started *after* both, because a movement
  goal can be driven by an animation that moves the root bone, and the choice of motion
  must exist before the mover is created.
- The whole assignment block is failure-guarded: any error thrown while projecting an
  action discards all script data, releasing the entity rather than leaving it half-driven.
  A rebuild without exceptions needs an equivalent bail-out, because a partly assigned
  action is worse than none.
- There is no `action_animation` callback here; that one fires from the animation
  completion callback instead, since animations end on their own schedule.

## `assign_movement(action) -> bool`

**Contract** — translates the action's movement demand into the entity's path manager.
Returns whether movement is still outstanding. Refuses, with a script error, on an entity
that has no movement machinery; silently completes on a dead one.

```text
FUNCTION assign_movement(action)
  IF movement already completed THEN RETURN false
  IF entity is alive-capable AND NOT alive THEN mark completed; RETURN false
  IF entity has no movement machinery THEN log error; RETURN true

  SELECT ON action.goal_type
    object          : path = level path
                      destination = target object's position and its level vertex
    patrol path     : path = patrol path
                      install the path, its start rule, its end rule, its randomness,
                      and, when given, the point to treat as already visited
    position        : path = level path
                      destination position as given; resolve it to a level vertex —
                      first by direct lookup, then, failing that, by walking from the
                      entity's own vertex toward the position until the walk leaves
                      the navigation mesh
    node position   : path = level path; the caller supplied the vertex, trust it
    free position   : path = level path, but bypass the navigation mesh entirely:
                      overwrite the detail path with the two points
                      (here, there) and mark the action complete once the entity is
                      within 0.2 of the destination
    anything else   : desired speed = 0; mark completed
  IF the path is fully resolved AND has been walked THEN mark completed
```

**Notes**

The "free position" case is the one that can move a creature off the navigation mesh; it
exists for scripted sequences where the authored destination is not walkable. The 0.2
arrival tolerance and the 0.1 path-changed tolerance are the only numbers in the function
and neither has a discoverable derivation — they read as tuned-by-feel.

The fallback vertex lookup matters: an authored position a few centimetres inside a wall is
common in shipped level data, and rejecting it would strand the creature.

## `assign_sound(action) -> bool`

**Contract** — owns the state machine for the action's one sound. Returns whether the sound
channel is still outstanding.

```text
FUNCTION assign_sound(action)
  IF sound already completed THEN RETURN false
  IF no sound object yet THEN
    IF the action names a sound THEN create it as an effect with the action's AI type
    ELSE mark the channel completed                   # nothing to play
  ELSE IF the sound is not currently audible THEN
    IF it has never started THEN
      play it at the bone-relative position the action asks for
      mark it started
    ELSE mark the channel completed                   # it started and has now finished
  RETURN NOT completed
```

**Invariants** — "not currently audible" is the completion test, so a sound that was
cancelled by the mixer's voice stealing completes its action rather than hanging it. The
sound's AI-perception type comes from the action, so a scripted sound is heard by other
creatures exactly as an engine-emitted one would be.

## `assign_particles(action) -> bool`

**Contract** — the same two-step start-then-complete machine for the action's particle
system, except the system is created when the action is built rather than here. A particle
action with no system completes immediately. The system is positioned from the bone-relative
transform before it is started, so the first emitted frame is already in the right place.

## `assign_animation(action) -> bool`

**Contract** — resolves the action's animation name into a motion on the entity's skeleton
and records it as the one to play next, along with whether the animation is allowed to
drive the entity's position. A name that resolves to nothing leaves the channel
outstanding but plays nothing — a *safe* lookup, because shipped scripts name animations
that some models do not have.

## `start_script_animation` — playing the resolved motion

**Contract** — starts the resolved motion on *every* bone group of the skeleton, so the
scripted animation overrides whatever the creature's own animation layer wanted. Returns
whether an animation is playing. Does nothing when the resolved motion is already the one
playing, which is what makes this safe to call every frame.

```text
FUNCTION start_script_animation()
  IF under control AND no current action AND queue not empty THEN process()
  IF NOT (under control AND current action has an unfinished, named animation) THEN
    forget the playing motion
    RETURN false
  IF resolved motion == playing motion THEN RETURN true      # already running
  playing motion = resolved motion
  FOR EACH bone group
    blend = play the motion on this group, looping
    IF this is the first group that accepted it THEN
      attach the completion callback to this blend only
      IF the action wants animation-driven movement THEN
        create a mover from this blend
  RETURN true
```

**Invariants**

- The completion callback is attached to exactly one blend, the first that accepted the
  motion. Attaching it to all of them would fire the action's completion once per bone
  group.
- The animation is started as looping; it is the callback, not the loop, that ends it.

## `on_animation_complete` — the animation callback

**Contract** — fired by the animation layer when the scripted motion's blend finishes.
Fires the `action_animation` script callback, marks the animation channel complete, forgets
the playing motion, and pumps the queue so the next action starts in the same frame.
Ignores blends that belong to a bone group rather than the whole body, so a partial-body
blend cannot end a whole action.

## `on_animation_frame` — the per-frame visual callback

**Contract** — installed while under script control. Each frame, if an action is current,
it re-positions the action's sound and particle system from their bone-relative transforms,
so both follow a moving skeleton. Does nothing at all when the queue is empty.

## `updated_matrix(bone_name, position_offset, angle_offset) -> transform`

**Contract** — builds the world transform for an attachment: the offset rotation and
translation, then, if a bone was named, composed with that bone's current pose and the
entity's own transform. With no bone named the result is the raw offset, i.e. a
*world-space* placement rather than an attached one. This is the single piece of geometry
in the file and both the sound and the particle channels depend on it.

## `finish_action(action)`

**Contract** — releases what the action acquired: destroys the current sound outright, and
destroys the particle system unless the action marked it self-removing, in which case the
particle layer will reclaim it when it stops emitting. Called exactly once per action, on
retirement or on queue clear.

## `clear_action_queue`

**Contract** — finishes the front action, then destroys every queued action and forgets the
animation state. Leaves the entity wherever it happened to be — it does not stop movement
or restore the creature's own behaviour; releasing script control does that.

## `sound_callback` and `process_sound_callbacks`

**Contract** — the senses layer reports heard sounds at a moment when calling into script
is unsafe, so `sound_callback` only *records* the event (emitter identity, AI sound type,
position, power) and only when the object actually has a sound callback installed.
`process_sound_callbacks` later replays every recorded event into script and clears the
list. Splitting them is the decision; the recorded tuple is the wire between the halves.

**Notes** — the emitter is recorded as an entity identifier rather than a reference,
because the emitter may be destroyed between the event and its replay.

## `check_object_visibility(object)` and `check_type_visibility(section)`

**Contract** — ask the entity's visual memory whether a particular object is visible right
now, or whether *any* currently visible object was spawned from the named configuration
section. Both answer false on an entity with no memory. The second is a linear scan of the
visible set and is the only way a script can ask "do I see a dog" without naming one.

## Lifecycle hooks

**Contract** — `init` and `reinit` reset the queue and re-arm capturability; `net_spawn`
marks the mixin live and forces the entity visible and enabled; `net_destroy` marks it not
live; `shedule_update` pumps the queue but only while under script control; `update_client`
re-asserts the scripted animation every frame. Construction resolves the two identities the
mixin needs — the entity itself, and the same entity viewed as a creature with movement and
memory, which may be absent.

## `patrol_path_name`

**Contract** — the name of the patrol path of the *last* queued action, not the current one;
empty when the queue is empty. Returning the back rather than the front is deliberate —
scripts ask this to find out where the creature is ultimately headed.

## `current_enemy`, `current_corpse`, `enemy_strength`

**Contract** — always answer "nothing" at this level. They exist so the script-visible
facade can call them on any entity; creature types that have enemies override them.
