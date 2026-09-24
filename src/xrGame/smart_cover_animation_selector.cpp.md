# src/xrGame/smart_cover_animation_selector.cpp

> Runs one planning cycle per clip boundary, turns the resulting action into a motion the model can play, and reports the clip's third marker back to the action as the moment its effect lands.

**Needs** — [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`Inventory.h`](Inventory.h.md) · [`HudItem.h`](HudItem.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — reached through its declarations in [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md); callers name that, not this file.
**Tier floor** — T1: reads playback time and motion markers out of the renderer's animation state

## Purpose

The clock of smart-cover behaviour. Everywhere else in the AI the plan advances on a
scheduler tick; here it advances on *clip boundaries*, because every smart-cover action is
an animation and nothing may be decided in the middle of one.

Two events come out of the animation layer and this file turns both into plan events: a
clip has ended (run a cycle, pick the next clip) and a marker inside a clip has been
crossed (tell the running action its moment has arrived — the frame a shot is fired, a
creature clears the wall, a posture completes).

## State

Declared in
[`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md).

```text
CONSTANT animation_speed_factor = 1.0      # console-tunable global playback scale
```

## `initialize`

**Contract** — initializes the planner, then **forces an animation update immediately** and
arms the clip-boundary flag so the first query runs a planning cycle.

**Invariants** — the forced update is not an optimization. Smart-cover actions install bone
callbacks, and a bone callback can be invoked by a collision raycast *before* the frame's
normal animation update has run, at which point the blends those callbacks need do not yet
exist. Updating here guarantees they do. A rebuild whose animation state is always valid
does not need this; one that defers blend evaluation to a frame phase must reproduce the
guarantee some other way.

## `finalize`

**Contract** — finalizes the planner, and does nothing if the planner was never
initialized. The guard matters because a creature can be pulled out of a cover before it
ever got a planning cycle.

## `select_animation` — the central query

**Contract** — asked by the animation layer for the clip to play. Always reports that the
clip owns the creature's movement. Two paths: at a clip boundary it advances the plan and
selects a new clip; otherwise it re-reports the current clip and checks whether a marker
was crossed since the last query.

```text
FUNCTION select_animation() -> (motion, animation_owns_movement = true)
  IF a clip boundary is pending
    IF the planner is running
      tell the current action its animation ended
      clear the pending flag; reset the playback stamp
      IF the planner stopped              RETURN the creature's global animation
    run one planning cycle
    IF the planner stopped                RETURN the creature's global animation
    tell the current action "no marker this query"
    IF the current action is not animated RETURN the creature's global animation
    ask the current action for its clip name
    REQUIRE the creature is in a cover
    RETURN the motion for that clip name
  # --- mid-clip ---
  motion = the motion for the selected clip name
  IF this is the first query
    clear the first-query flag; reset the playback stamp
    tell the current action "no marker"
    RETURN motion
  blend = the creature's currently playing global blend
  IF there is none
    reset the playback stamp; tell the action "no marker"
    RETURN motion
  IF the clip has fewer than three markers
    tell the action "no marker"
    RETURN motion
  window = (clamped previous stamp, blend playback time + 0.1)
  store the new stamp
  IF the third marker lies in the window
    tell the current action "marker"
  ELSE
    tell the current action "no marker"
  RETURN motion
```

**Invariants** —

- **Exactly one planning cycle per clip.** The pending flag is cleared before the cycle
  runs, so a cycle that itself ends the plan cannot trigger a second.
- **The planner can stop mid-query**, twice: the action's end-of-animation handler may
  finish the plan, and so may the planning cycle. Both are checked, and both fall back to
  the creature's ordinary global animation — leaving a cover must not leave the model
  frozen.
- **The first two markers of a clip are footsteps.** The third is the smart-cover event.
  That is a convention shared with the animators and it is frozen in the shipped motion
  data; a clip with fewer than three markers simply has no event. A rebuild that names
  markers instead of indexing them is better, but it must then re-author every shipped
  clip's marker set.
- **The marker window is half-open and forward-looking**: it runs from the previous
  query's stamp to the current playback time *plus a tenth of a second*. The lookahead
  fires the event slightly early, which is what makes a shot look simultaneous with the
  animation rather than a frame late.
- **The previous stamp is clamped up to the current time** before use, because playback
  time can be reset to zero while a clip is still playing. Without the clamp the window
  would invert at the reset and the marker would fire spuriously or not at all.

**Notes** —

- The clip name is selected by the action, not here; this file only resolves the name to a
  motion. Resolution is by exact name and fails loudly on a missing clip — an authored
  cover naming a clip the model does not have is an authoring error.
- A cover that does not permit firing skips a whole branch. In the original that branch —
  disabled in shipping builds — appended the active weapon's animation-slot number to the
  clip name and fell back to slot two when no such clip existed. The surviving requirement
  is that a cover permitting fire may need *weapon-specific* variants of the same clip, and
  the fallback slot is the generic rifle. A rebuild that keeps one clip per action for all
  weapons matches the shipped behaviour; one that reinstates the variants must ship the
  variant clips too.

## `on_animation_end`

**Contract** — told by the animation layer that the playing clip finished. Sets the pending
flag and nothing else; the work happens at the next query. Asserts the flag was not already
set — two ends without an intervening query means the plan has missed a boundary.

## `modify_animation`

**Contract** — called when a blend starts; rescales its playback speed to the motion's
authored speed times a global factor. Does nothing if there is no blend.

**Notes** — the factor is a console-tunable global defaulting to one, i.e. no change. It
exists so the whole smart-cover system's pacing can be tuned at once during development.
Because it multiplies the *authored* speed rather than replacing it, per-clip pacing is
preserved.

## `save` / `load` / `setup`

**Contract** — all three forward to the planner. The planner's state — which is the
creature's position in its cover rhythm — is therefore part of the save format, which is
what makes a creature reloaded in a cover resume rather than restart.
