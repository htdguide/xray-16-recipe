# src/xrGame/stalker_animation_manager_update.cpp

> The per-frame resolution: a strict three-tier priority ladder — scripted animation, whole-body animation, or the head/torso/legs trio — plus the speed feedback that makes a stalker's body travel at the pace its own leg animation asks for.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_manager_inline.h`](stalker_animation_manager_inline.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: runs for every live stalker every frame, and drives the skeleton

## Purpose

Everything else in the stalker's animation system chooses a motion for one channel. This
file decides *which channels get to choose at all*, and it does so with a ladder whose
strictness is the whole design: a scripted animation suppresses everything, a whole-body
animation suppresses the trio, and only when neither is playing do head, torso and legs run
together partitioned by bone group.

It is also where the loop closes between animation and movement: the legs pick a motion, the
motion has an authored speed, and the body is then driven at that speed rather than the
other way round. A stalker's feet do not slide because the feet are in charge.

## `update`

**Contract** — advance the whole animation system by one frame. Catches and contains any
fault from the resolution, reporting the offending stalker and resetting every channel
rather than propagating. Runs under a named profiler scope.

```text
FUNCTION update()
  TRY
    update_impl()
  ON ANY FAILURE
    report the stalker's visual name and identifier
    reset all five channels
```

**Notes** — swallowing faults here is a deliberate robustness choice with a real cost: a
model missing an animation the code indexes produces a stalker that silently stops animating
instead of a crash, and the report is the only evidence. The alternative — failing hard — was
what the original did and was changed. A rebuild should keep the containment and make the
report loud, because a stalker frozen mid-stride is otherwise diagnosed as a physics bug.

## `update_impl` — the priority ladder

**Contract** — resolve every channel for this frame. Returns immediately for a dead stalker.

```text
FUNCTION update_impl()
  IF NOT alive  RETURN

  update_tracks()              # advance the skeleton's blends, if anything needs it
  play_delayed_callbacks()     # run last frame's animation-end callbacks, now that it is safe

  IF play_script()  RETURN     # tier 1: a script asked for a specific animation
  IF play_global()  RETURN     # tier 2: a whole-body motion is warranted

  play_head()                  # tier 3: the three channels run together,
  play_torso()                 #         partitioned by bone group
  play_legs()

  torso.synchronize(legs)      # align the upper body's phase to the stride
```

**Invariants**

- **Callbacks run before selection, and they run deferred.** An animation ending fires a
  callback into script or into whichever subsystem claimed the global channel, and that
  callback will typically queue a new animation or change the stalker's goal. Running it
  from inside the renderer's blend-end notification would mutate the channel that is being
  torn down; running it at the top of the next frame's update means the selection that
  follows sees the result. The one-frame delay is the cost and it is invisible.
- **Each tier resets the tiers below it.** A script animation resets global, torso and legs
  (and head, unless the head is its own bone group); a global animation resets torso and legs
  (and head, likewise). Without the reset the suppressed channels would keep their blend
  handles and resume from a stale state when the suppression lifted.
- **The torso is synchronized to the legs, not the reverse.** The upper body's motion is
  phase-matched to the stride so that the weapon does not bob against the walk cycle. The
  legs are the reference because they are what the ground contact is authored against.

## `update_tracks`

**Contract** — advance the skeleton's blend tracks, but only if some channel has work. See
`non_script_need_update` in
[`stalker_animation_manager_inline.h`](stalker_animation_manager_inline.h.md); the script
channel adds one more condition — a queued animation *and* a script callback registered for
it — because a queued animation with nobody listening for its end does not need per-frame
progress tracking.

**Notes** — this is the frame budget's main lever for a crowd of stalkers. A stalker standing
still with nothing pending skips the whole track advance.

## `play_script`

**Contract** — if the script queue is non-empty, play its front entry on the script channel
and report that tier 1 claimed the frame. Otherwise reset the channel and report false.

```text
FUNCTION play_script() -> bool
  IF the queue is empty
    clear the start-new-animation flag
    reset the script channel
    RETURN false

  clear the aiming bone callbacks if they are in blend mode
  reset global, torso and legs                 # and head, unless the head is its own group

  entry = the front of the queue
  script.animation = entry.motion
  IF entry drives the movement controller
    script.target_matrix = entry.transform relative to the stalker
    IF this is the first frame of this entry
      destroy any existing movement controller, so the new one starts from this transform
  play it on the script channel, honouring the entry's movement-controller and
    local-space flags, restricted to the script bone-group mask
  IF the head is its own bone group
    also select and play a head motion underneath
  RETURN true
```

**Invariants** — the movement controller is destroyed and rebuilt only on the *first* frame
of an entry that has a transform. A scripted animation with a transform moves the stalker's
body along a path baked into the motion, and that path is anchored to the transform at the
moment it starts; rebuilding it every frame would re-anchor it every frame and the stalker
would never advance.

**Notes** — the bone-group mask restricts a script animation to everything except the head's
own group when that path is compiled in, which is what lets a scripted gesture play while
the stalker keeps looking at whoever it is talking to. With the path off, the mask is
ignored and the script animation takes the whole body including the head.

## `play_global`

**Contract** — ask the global channel for a motion; if it names one, play it on the whole
body and report that tier 2 claimed the frame. Otherwise clear the aiming callbacks, reset
the channel and report false.

```text
FUNCTION play_global() -> bool
  (motion, wants_movement_controller) = assign_global_animation()
  IF motion is none
    clear the aiming bone callbacks if they are in blend mode
    reset the global channel
    RETURN false

  reset torso and legs                        # and head, unless the head is its own group
  global.animation = motion
  play it, honouring the movement-controller request
  IF a global modifier hook is installed, let it adjust the resulting blend
  IF the head is its own bone group
    also select and play a head motion underneath
  RETURN true
```

**Notes** — the modifier hook runs *after* the blend exists, which is the only time it can
work: it adjusts speed, weight or phase on a live blend. That is why it is a third hook and
not folded into the selector.

## `play_legs` — the speed feedback loop

**Contract** — select and play the leg motion, and drive the stalker's body speed from the
blend weight of the motion that is actually running. Reads and writes the manager's speed
state.

```text
FUNCTION play_legs()
  first_time = the legs channel has no animation yet
  changed    = legs.animation(assign_legs_animation())    # true if the motion changed

  # An unchanged motion mid-blend: carry the previous speed forward along the blend,
  # so that a speed read next frame starts from where this frame actually was.
  IF NOT first_time AND NOT changed AND a blend exists
    previous_speed = interpolate(previous_speed -> target_speed, by blend weight)

  play the leg motion, marked as moving when the target speed is non-zero

  # A changed motion: the body's speed is the blend between the old speed and the new
  # motion's, weighted by how far the crossfade has come.
  IF changed AND a blend exists
    speed = interpolate(previous_speed -> target_speed, by blend weight)
    IF speed is non-zero
      tell the movement system to travel at this speed
```

**Invariants** — the target speed is set by the leg *selection*, in
[`stalker_animation_legs.cpp`](stalker_animation_legs.cpp.md), which is what makes this a
loop rather than a pair of independent systems. The movement system asks for a movement type
and a direction, the leg channel picks the motion authored for it, the motion's own speed is
handed back, and the body moves at that speed. A rebuild that drives the animation from the
body's speed instead will get foot sliding on every gait change, because the authored speeds
are not a continuum.

Interpolating along the blend weight during a crossfade, rather than switching speed with the
motion, is what makes a walk-to-run transition accelerate smoothly. The two branches cover
the two cases — same motion still blending in, and a new motion just started — and both read
the same blend weight.

## `play_head`, `play_torso`

**Contract** — select and play the head and torso motions on their channels, with no
movement controller and no special flags. Both are one line; the selection is the work, and
it lives in [`stalker_animation_head.cpp`](stalker_animation_head.cpp.md) and
`stalker_animation_torso.cpp`.

## `play_delayed_callbacks`

**Contract** — run at most one deferred callback: the script-animation callback if one is
pending, otherwise the global-channel callback. Clears the flag before calling.

**Invariants** — the flag is cleared **before** the call, because the callback frequently
queues another animation whose end will set the flag again. Clearing afterwards would erase
that.

At most one runs per frame, and the script callback wins. Both firing in one frame would mean
a script callback and a subsystem callback mutating the channel set in the same update, with
the second seeing the first's half-applied result.
