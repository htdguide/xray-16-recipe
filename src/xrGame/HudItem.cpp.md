# src/xrGame/HudItem.cpp

> Anything the player holds in their hands: the state machine every held item runs, the animation whose *end* drives the next state, the sound bank keyed by alias, and the two transforms that put a world position into the first-person view's own projection.

**Needs** — [`HudItem.h`](HudItem.h.md) · [`HudSound.h`](HudSound.h.md) · [`player_hud.h`](player_hud.h.md) · [`physic_item.h`](physic_item.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`Inventory.h`](Inventory.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`actor_defs.h`](actor_defs.h.md) · [`inventory_space.h`](../xrServerEntities/inventory_space.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../xrCore/Animation/SkeletonMotions.hpp.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a state machine, timers and matrix composition; the animation and sound handles are interfaces

## Purpose

Every weapon, grenade, detector, medkit and torch the player can hold derives from this.
It supplies four things, and the first two are the load-bearing ones.

**A state machine whose transitions are driven by animation completion.** An item is
hidden, idle, showing, hiding or bored; a state change plays a motion, and the *end of
that motion* fires the transition to the next state. Nothing polls. Subclasses extend the
enumeration past the base's last value and inherit the mechanism unchanged. This is why
weapon handling in this game feels weighty: the code cannot skip ahead of the animation.

**A state machine that is driven through the network even in single player.** Switching
state does not set the state; it sends an entity event, which comes back and sets it.
Single player runs both sides in one process, so this is a same-frame round trip — but it
means a remote client and the server agree on the item's state by construction. A rebuild
that "optimizes" the local case into a direct call has silently forked the two paths.

**A sound bank keyed by alias** rather than by field, so that a subclass adds a sound by
naming a configuration key and an alias and never touches the playing code.

**Two transforms** from world space into the first-person view's own space, which exists
because the held model is rendered with a *different field of view and near plane* from
the world.

## State

```text
RECORD HudState                    # the base of every held item
  state          : int   # hidden, idle, showing, hiding, bore, or a subclass's own
  next_state     : int   # what the current animation is transitioning toward
  state_since    : int   # when the current state was entered
  substate_since : int   # a subclass-resettable clock within a state

RECORD HudItem
  hud_section     : optional<text>   # the first-person model's configuration section
  animation_slot  : int              # which of the player's hands/slots this occupies
  sounds          : SoundBank        # alias -> one or more sounds
  pending         : bool             # a transition is in flight; refuse new commands
  render_hud      : bool             # draw the first-person model at all
  inertion_enabled, inertion_allowed : bool   # the held model's sway
  current_motion       : optional<text>
  current_motion_def   : optional<MotionDefinition>  # its marks and length
  motion_start, motion_end, motion_now : int
  motion_state    : int    # the state the running motion belongs to
  random_variant  : int    # which variant of a multi-variant motion was chosen
  watching_for_end: bool   # whether the motion's end should fire a transition
```

Invariants:

- A motion is being watched exactly when `watching_for_end` is set, and then the three
  timestamps are consistent and ordered. Stopping a motion without a callback clears all
  four together — that is the whole purpose of the "without callback" variant, and it
  exists so that an item being put away does not fire the transition its animation would
  otherwise have fired.
- `random_variant` is chosen when a motion starts and is reused to pick the matching
  *sound* variant, so a weapon whose reload has three takes plays the take that matches
  the animation.
- The item knows itself as three things at once — a physical object, an inventory item and
  a held item — and captures the first two at construction. Every later access asserts
  they are present.

## `Load`

**Contract** — reads the first-person model's configuration section (optional; an item
with none is never drawn in the player's hands), the animation slot, and the idle-fidget
sound. The slot is required when the class has no hard-coded default and optional when it
does.

## The state machine — `SwitchState`, `OnEvent`, `OnStateSwitch`, `OnAnimationEnd`

**Contract** — `SwitchState` is a *request*: it records the intended next state and, on
the authoritative side, sends a state-change event naming it. The event handler applies
it through `OnStateSwitch`, which stamps the state, resets the clocks, and runs whatever
the new state entails. When a watched motion ends, `OnAnimationEnd` fires a script
callback and then may request the next state.

**Invariants** — the request is refused outright on a client mirror: a client never
decides its own item's state. On the authoritative side the event is sent only if the
object is local and not being destroyed, and the source's own comment stresses that
exactly one event per state is sent — a duplicate would play the animation twice.

A remote item additionally copies the applied state into the intended state, because it
has no pending transition of its own to preserve.

```text
FUNCTION switch_state(target)
  IF this is a client mirror THEN RETURN          # clients do not decide
  next_state = target
  IF the object is local and not being destroyed THEN
    send a state-change event carrying target     # comes back through on_event

FUNCTION on_state_switch(new_state, old_state)
  state = new_state ; reset both clocks
  IF remote THEN next_state = new_state
  IF new_state is bore THEN
    pending = false
    play the fidget animation
    play the fidget sound at the held model's position, using the same random variant

FUNCTION on_animation_end(state)
  IF the holder is the player THEN
    fire the script animation-end callback with the item, its hud section,
      the motion name, the state and the slot
  IF state is bore THEN request idle
```

**Notes** — the base handles exactly one state, the idle fidget. Everything else is the
subclass's; what the base contributes is the *shape*, and the shape is the contract.

## `UpdateCL` — the motion watcher

**Contract** — once per frame, while a motion is being watched: scan the motion's
**marks** for any whose interval the frame just entered, and fire the mark callback for
each; then, when the motion's end time has passed, clear the watch and fire the
animation-end transition.

**Invariants** — a mark fires on the *edge*: it is reported when the previous frame's time
was outside its interval and this frame's is inside. That makes marks fire exactly once
regardless of frame rate, and it is why the previous frame's motion time is kept rather
than only the current one.

```text
FUNCTION update_per_frame()
  IF no motion is being watched THEN RETURN
  previous = (motion_now - motion_start) in seconds
  current  = (now - motion_start) in seconds
  FOR EACH mark IN the motion's marks
    IF the mark was not active at `previous` and is active at `current` THEN
      fire the mark callback with the motion's state and the mark
  motion_now = now
  IF motion_now > motion_end THEN
    clear the watch and the three timestamps
    fire the animation-end transition for the motion's state
```

**Notes** — marks are the animation-driven game events: the frame a casing ejects, a
magazine drops, a footstep sounds. They come from the animation data, so the *content*
carries the timing and a rebuild must read the marks out of its motion format rather than
hard-coding delays.

## `PlayHUDMotion` and its variants

**Contract** — three forms. The plain form plays a motion and arms the end watch, stamping
the start and end times from the motion's reported length; it returns that length, or zero
when the motion does not exist, in which case no watch is armed. The two-name form tries
the first name and falls back to the second. The no-callback form plays the motion without
arming the watch, and is also the *only* path that actually reaches the animation system.

**Invariants** — the no-callback form has a second job: when the item is **not** currently
attached to the player's hands, it does not play anything — it merely asks how long the
motion *would* be. That is how a weapon in a non-player's hands, or one whose first-person
model is not loaded, still runs its state machine on the right schedule. The state machine
is therefore driven by animation *durations*, not by animation playback, and works
identically whether or not anything is drawn.

```text
FUNCTION play_motion(name, mix_in, state) -> int
  length = play_motion_no_callback(name, mix_in)
  IF length > 0 THEN
    watching_for_end = true
    motion_start = now ; motion_now = now ; motion_end = now + length
    motion_state = state
  ELSE watching_for_end = false
  RETURN length

FUNCTION play_motion_no_callback(name, mix_in) -> int
  current_motion = name
  IF the item is attached to the player's hands THEN
    RETURN the hands' play(name, mix_in), which also yields the motion definition
           and which random variant it chose
  ELSE
    random_variant = 0
    RETURN the length this motion would have, from the shared animation bank
```

## The idle family — `PlayAnimIdle`, `TryPlayAnimIdle`, `PlayAnimIdleMoving`, `PlayAnimIdleMovingCrouch`, `PlayAnimIdleSprint`, `PlayAnimBore`

**Contract** — the idle animation is chosen from the holder's *movement state*, in a fixed
priority: sprinting, then moving-while-crouched, then moving, then standing. Each variant
is used only if it actually exists in the data; a missing variant falls back to the plain
idle, and sprint falls back through a second spelling before that.

**Invariants** — the fallback chain is checked against the data at every play, not at
load. That is what lets one item's configuration supply a sprint idle and another's not,
with no per-item flag.

**Notes** — every animation name is tried in two spellings, one with an `anm_` prefix and
one with `anim_`. Two generations of the game data named these differently and both ship.
This is pure compatibility, and a rebuild reading the original data must reproduce it.

## `OnMovementChanged`

**Contract** — when the holder starts sprinting or starts moving, and the item is idle
with no animation running, re-pick the idle animation and reset the substate clock. Does
nothing mid-animation, so a reload is never interrupted by starting to run.

## `isHUDAnimationExist` / `WhichHUDAnimationExist`

**Contract** — whether a named motion exists for this item. When the item is in the
player's hands, the question is asked of the loaded hand motions; otherwise it is asked of
the shared bank by section, and a motion shorter than a tenth of a second counts as
absent. Reports the first of two names that exists, or none.

**Invariants** — in the first-person path the name is **suffixed for widescreen** when the
item occupies a particular attachment slot. The game ships separate hand animations for
wide and narrow aspect ratios, because the model is framed differently; a rebuild must
carry that suffix or the weapon will be half off screen.

**Notes** — the "shorter than a tenth of a second means absent" rule is how a missing
motion is distinguished from a real one in the shared bank, which returns zero-length for
unknown names. It is a sentinel, not a threshold.

## `renderable_Render`

**Contract** — decide whether the *world* model of the item should be drawn this frame.
It is drawn when the item has no parent (lying on the ground), when the first-person
model is not being rendered and the item is not hidden, or when the item is *attached* to
its owner's body — a holstered weapon on a character's back. It is never drawn while the
first-person model is.

**Invariants** — the empty branch for "the first-person model is being drawn" is
deliberate: the held model is drawn by the player's hands system, not by the item.

## `TransformPosFromWorldToHud` / `TransformDirFromWorldToHud`

**Contract** — map a world position or direction into the coordinates the first-person
model is rendered in, so that something computed in the world (a laser dot, a scope's aim
point) can be drawn against the held model.

**Invariants** — the first-person view has its own field of view, its own near plane and,
when the player is in first person, its own camera transform. The mapping is a *round
trip*: into the hands' view and projection, then back out through the world's projection
and view inverses, so the result is a world-space point that projects to the same pixel
the hands' projection would have put it at.

```text
FUNCTION world_to_hud(point)
  view = the world view, replaced by the hands' camera when in first person
  hud_projection = projection from the hands' field of view, the hands' near plane,
                   and the weather's far plane
  point = view applied to point
  point = hud_projection applied to point
  point = inverse of the world projection applied to point
  point = inverse of view applied to point
```

**Notes** — the far plane comes from the *weather*, not from the renderer, which keeps the
held model's depth range consistent with the world's fog. The hands' field of view is a
global scaled by the world's, so a zoomed world view also narrows the hands.

## `HudItemData`

**Contract** — the first-person attachment this item currently occupies, or none. The
player's hands hold at most two attachments and the item is whichever of them names it as
its parent.

**Invariants** — "none" is the normal case, not an error: an item in a non-player's hands,
in a container, or on the ground has no attachment, and every caller must handle it. This
is the single test that separates "this item is being held by the player right now" from
every other situation.

## Activation, deactivation and parentage — `ActivateItem`, `DeactivateItem`, `SendDeactivateItem`, `SendHiddenItem`, `OnMoveToRuck`, `OnH_A_Chield`, `OnH_B_Chield`, `OnH_B_Independent`, `OnH_A_Independent`, `on_a_hud_attach`, `on_b_hud_detach`

**Contract** — drawing and putting away, and the four parentage transitions. Moving an item
to the rucksack switches it to hidden. Becoming a child stops the current animation without
firing its callback. Becoming independent stops every sound, refreshes the transform, and
detaches the first-person model. Re-attaching the first-person model *resumes the motion
that was running*, without a callback, so that picking the view back up mid-reload does not
restart or complete it.

**Invariants** — the resume is the reason the state machine is driven by durations rather
than by playback: the motion's end time was set in world time when it started, so it
expires on schedule whether or not it was visible for all of it.

## `PlaySound`

**Contract** — play a sound by alias at a position, from the item's bank, attributed to
the holder's root object, in two-dimensional mode when the item is in the player's hands.
A second form selects a specific variant rather than a random one.

**Invariants** — two-dimensional playback for a held item is what makes the player's own
weapon sound centred rather than positioned; the same sound from another character's
weapon is positional.

## `GetHUDmode`

**Contract** — true only when the item's holder is the player, the player is in
first-person view, and the item is actually attached to the hands. The single authority
used throughout for "am I being drawn and heard as the player's own item".

## `UpdateHudAdditonal`, `UpdateXForm`, `on_renderable_Render`, `GetInertionFactor`, `GetInertionPowerFactor`, `render_hud_mode`, `render_item_3d_ui`, `render_item_3d_ui_query`, `CheckCompatibility`, `GetCurrentHudOffsetIdx`, `MovingAnimAllowedNow`, `need_renderable`, `Action`, `OnMotionMark`, `OnActiveItem`, `OnHiddenItem`

**Contract** — the extension points. Two are pure and must be answered by every subclass:
recomputing the item's transform, and drawing its world model. The rest default to
"nothing", "yes" or "one" and exist so that a subclass can add sway, an attached
interface, a compatibility rule with another held item, or a response to an animation
mark, without the base knowing about any of them.
