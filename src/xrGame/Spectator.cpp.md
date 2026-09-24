# src/xrGame/Spectator.cpp

> The entity a player becomes when he is dead, has not joined yet, or is watching a recorded match: a bodiless camera with five viewing modes and a rule set saying which of them this game mode will allow.

**Needs** — [`Spectator.h`](Spectator.h.md) · [`Actor.h`](Actor.h.md) · [`CameraLook.h`](CameraLook.h.md) · [`spectator_camera_first_eye.h`](spectator_camera_first_eye.h.md) · [`EffectorFall.h`](EffectorFall.h.md) · [`Level.h`](Level.h.md) · [`Inventory.h`](Inventory.h.md) · [`HudItem.h`](HudItem.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`map_manager.h`](map_manager.h.md) · [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md) · [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md) · [`xrServerEntities/xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [`xrEngine/IInputReceiver.h`](../xrEngine/IInputReceiver.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: camera arithmetic, input handling and entity bookkeeping

## Purpose

A multiplayer player who is dead must still see the match, and a recorded match must be
watchable from any angle. Both are served by making the viewer a real entity — it spawns
from a server record, it has an identity, it is what the level considers the current view
entity — that simply has no body and no inventory. That decision is why so much of the
file is bookkeeping: the spectator is a first-class entity and every system that expects
the view entity to be an actor has to be told otherwise.

Three things in it are genuinely interesting and everything else follows from them:

1. **It keeps its own clock.** A spectator must move while the simulation is paused,
   because demo playback pauses the world and lets the viewer fly around it. So it
   measures its own elapsed time and never uses the engine's frame delta for movement.
2. **It drives somebody else's update.** While paused, the spectator manually advances the
   player it is watching, so that a paused demo still shows a consistent pose and heads-up
   display rather than a frozen half-updated frame.
3. **Which cameras are available is a game-mode decision, not a player one.** A team
   deathmatch may forbid free flight (it would be reconnaissance) and forbid watching the
   other team. The spectator asks the mode before every switch.

## State

```text
ENUM CameraMode = free_fly | first_eye | look_at | free_look | fixed_look_at

RECORD Spectator EXTENDS GameObject, InputReceiver
  cameras       : map<CameraMode, Camera>   # one instance each, all five built at construction
  active        : CameraMode
  last_mode     : CameraMode                # the non-free-fly mode to return to
  watching      : optional<Actor>           # whose view we are borrowing
  watch_index   : int                       # cursor into the candidate list
  last_watched_name : text                  # so a target survives its own respawn
  own_delta     : real                      # smoothed, paused-proof frame time
  timer         : Timer                     # feeds own_delta
  saved_camera_inertia : real
```

**Invariants**

- All five cameras exist for the spectator's whole life; switching only changes which one
  is asked for a transform. Building them lazily would cost a frame of wrong orientation
  on each first switch.
- The spectator's own position is always the active camera's position **lowered by the
  standard eye height**. The camera is the truth; the entity's transform is derived from
  it so that anything asking where the spectator is gets a sensible ground-level answer.
- A watched player reference is cleared the instant that player is destroyed. Nothing may
  hold a reference to a departing entity past its teardown.

## `UpdateCL`

**Contract** — the spectator's per-frame work: advance its own clock, pump the watched
player if the world is paused, decide whether the current watch target is still valid,
and drive the camera. Runs only on the machine whose player this spectator belongs to, or
during demo playback.

```text
FUNCTION UpdateCL()
  base.UpdateCL()

  # --- own clock ---
  measured  <- timer.elapsed_seconds() ; timer.restart()
  own_delta <- 0.3 * own_delta + 0.7 * measured     # smoothed: see note
  own_delta <- clamp(own_delta, small_positive, 0.1)

  # --- keep the watched player alive-looking while the world is frozen ---
  IF world is paused AND watching EXISTS THEN
    WITH engine frame delta forced to zero:
      watching.UpdateCL()
      watching.scheduled_update(0)
      game_mode.scheduled_update(0)

  IF multiplayer AND (this spectator is the local player OR playing a demo) THEN
    IF active != free_fly THEN
      IF watching EXISTS AND watching IS dead THEN switch_to(free_look)
      IF watching IS none THEN
        restore_previous_watch_target()
        IF watching EXISTS THEN switch_to(last_mode)
    IF this spectator is the current view entity THEN drive_camera(watching)
    RETURN

  # single player / fallback: walk the team hierarchy for the indexed player
  IF this spectator is the current view entity THEN
    IF active != free_fly THEN
      candidate <- the watch_index-th actor found by walking
                   team -> squads -> groups -> members of the local player's team
      IF candidate EXISTS THEN drive_camera(candidate) ; RETURN
      watch_index <- 0                       # index was stale
      IF the walk found nobody at all THEN switch_to(free_fly)
    drive_camera(none)
```

**Invariants**

- The smoothing of the measured frame time is a first-order filter weighted toward the new
  sample. It exists because free-flight movement multiplies this number directly, and an
  unsmoothed measurement makes the camera stutter on every frame-time spike. The upper
  clamp of a tenth of a second means the camera behaves as though the frame rate never
  drops below ten — a longer frame simply moves the camera less than real time would, which
  is far better than a lurch.
- The paused pump forces the engine's frame delta to zero around the calls. The watched
  player's update must recompute its pose and heads-up display from its *current* state
  without advancing any timer, animation or timeout. Getting this wrong makes a paused
  demo slowly play forward.
- A dead watch target drops the viewer to the free-look mode rather than to free flight —
  the viewer stays near the action instead of being dumped into the level at large.

**Notes** — the two paths through this routine, the multiplayer one and the team-hierarchy
walk, do the same job by different means; the walk is the older mechanism and is reachable
only outside multiplayer, where there is nothing to spectate. A rebuild should keep the
candidate-list mechanism and delete the walk.

## `cam_Update`

**Contract** — produces the frame's camera transform. Given a player to watch, the active
mode decides how that player's own camera is translated into the spectator's; given
nobody, the free-flying camera (or the fixed one) is simply updated in place. Either way
the free-flying camera is kept synchronized with wherever the view actually is, and the
spectator's entity position is set from the camera.

```text
FUNCTION cam_Update(target)
  IF target EXISTS THEN
    cam <- cameras[active]
    CASE active OF
      first_eye:  cam.set_from(target.active_camera)          # exactly his eyes
      look_at:    cam.set(yaw: target.camera.yaw,
                          pitch: target.camera.pitch,
                          roll: -target.transform.roll)       # orbit, level with his body
      free_look:  cam.parent <- target
                  focus <- target.position raised by eye_height
                  IF target IS dead THEN focus <- the same point taken through
                                                  a translation-only frame   # see note
                  cam.update(focus)
    # keep the free-flight camera where the view is, so switching to it does not jump
    cameras[free_fly].set_from(cam) ; cameras[free_fly].roll <- 0
    self.position <- cam.position lowered by eye_height
  ELSE
    cam <- cameras[fixed_look_at] IF active = fixed_look_at ELSE cameras[free_fly]
    cam.update(focus: self.position raised by eye_height)

  IF world is paused THEN
    WITH engine frame delta temporarily set to own_delta:
      camera_manager.adopt(cam)               # see note
  ELSE
    camera_manager.adopt(cam)
```

**Invariants**

- Every mode ends by writing the free-flight camera's transform. The free-flight camera is
  the *fallback*, and the viewer can drop into it at any moment — when his target dies,
  when he presses the key, when a target is destroyed. It must always already be where he
  is looking, or every fallback is a teleport.
- The eye height is applied consistently in both directions: the camera looks from a point
  raised above the entity, and the entity is placed a corresponding distance below the
  camera. The two must use the same constant.

**Notes** — the dead-target branch in the follow mode routes the focus point through a
translation-only frame, discarding the corpse's rotation. A ragdolled body's transform
tumbles; following it faithfully makes the camera roll with the corpse. Taking only the
translation keeps the camera upright over a body that is spinning.

The paused branch lends the camera manager the spectator's own frame time instead of the
zero the engine reports. The camera manager interpolates its field of view over time, and
with a zero delta the field of view never converges — the source notes exactly this. A
rebuild whose camera interpolation is driven by a supplied delta rather than a global one
has no such workaround.

## `IR_OnKeyboardPress`

**Contract** — the spectator's controls. Ignored entirely for a spectator that is not
locally controlled.

| Command | Effect |
|---|---|
| accelerate | doubles the free-flight speed multiplier while held |
| camera 1 / 2 / 3 | from free flight only, picks a watch target and switches to first-eye, orbit or follow |
| fire | advances the watch cursor to the next candidate; in first-eye, also re-points the level's view at him |
| zoom | cycles to the next *permitted* camera mode |

The cycle is the interesting one:

```text
FUNCTION cycle_camera()
  IF not a multiplayer game THEN RETURN
  IF not playing a demo AND this spectator is not the local player THEN RETURN

  next <- (active + 1) modulo mode_count
  IF the local player is NOT a declared spectator THEN
    advance `next` until the game mode permits it,
      giving up when the last mode is reached          # see note
    IF none is permitted THEN RETURN

  IF next = free_fly THEN switch_to(free_fly) ; watching <- none
  ELSE
    IF watching IS none THEN restore_previous_watch_target()
    IF watching EXISTS THEN switch_to(next) ; last_mode <- next
```

**Invariants** — a *declared* spectator (a player who joined to watch, never to play) skips
the permission filter entirely and gets every camera. The filter exists to stop a dead
*participant* from scouting the map for his living team-mates, and that concern does not
apply to someone who is not playing.

A non-free-flight mode is never entered without a target; switching to "watch nobody from
behind" would leave a camera pointing at nothing.

**Notes** — the permission search stops when it reaches the last mode rather than when it
has tried every mode, so it never wraps back through the modes it skipped past. With the
shipped permission sets this is invisible. A rebuild should try each mode exactly once,
starting from the one after the current.

## `IR_OnKeyboardHold`

**Contract** — free-flight and follow movement, per frame while a key is held. Only the
two free modes move at all; the rest are anchored to a target.

Pitch, yaw and zoom commands are handed to the active camera, except that the follow mode
ignores left and right (its yaw comes from the orbit, not from strafing). Translation is
computed along the camera's own axes at the acceleration multiplier scaled by the
spectator's **own** frame time, and is applied only if free flight is permitted or the
player is a declared spectator.

**Invariants** — the permission is re-checked on movement, not only on mode entry. A mode
switch can be legal while the movement it enables is not: the viewer may sit in free
flight where he was dropped and look around, but not fly.

The strafe axis is derived as the cross product of the camera's up and forward vectors
each frame rather than being stored, so strafing stays correct as the camera rolls.

## `IR_OnMouseMove`

**Contract** — turns pointer deltas into camera rotation. The scale is the ratio of the
camera's current field of view to the default, times the user's sensitivity setting. The
vertical axis is scaled by three quarters relative to the horizontal and may be inverted
by a user setting.

**Invariants** — scaling by the field-of-view ratio is what makes a zoomed view turn
proportionally slower, so that the apparent angular speed on screen is constant. Without
it, aiming through a narrow field of view is uncontrollable.

**Notes** — the three-quarters factor on the vertical axis compensates for the display's
aspect ratio being wider than tall, so that an equal pointer movement produces an equal
*apparent* rotation in both axes. It is a constant here and should be derived from the
actual aspect ratio in a rebuild.

## `SelectNextPlayerToLook`

**Contract** — chooses whom to watch. Builds the list of watchable players, then either
**restores** the previously watched one by name or **advances** the cursor to the next.
Returns whether a target was found. Does nothing in single player.

```text
FUNCTION SelectNextPlayerToLook(advance) -> bool
  IF single player THEN RETURN false
  IF there is no local player record THEN RETURN false
  watching <- none

  candidates <- empty ; restore_index <- none
  FOR EACH player_record IN all_players
    IF player_record IS permanently dead THEN CONTINUE
    IF the mode restricts spectating to one's own team
       AND player_record.team != local.team
       AND local is not a declared spectator THEN CONTINUE
    actor <- the live object with that player's entity identifier
    IF actor IS none OR actor IS NOT an actor THEN CONTINUE
    IF player_record.name = last_watched_name THEN restore_index <- candidates.count
    candidates.append(actor)

  IF NOT advance THEN
    IF restore_index EXISTS THEN watching <- candidates[restore_index] ; RETURN true
    RETURN false

  IF candidates IS empty THEN RETURN false
  watch_index <- watch_index modulo candidates.count
  watching <- candidates[watch_index]
  last_watched_name <- name of the player behind `watching`
  RETURN true
```

**Invariants**

- The target is remembered **by player name**, not by entity identifier. A player who dies
  and respawns is a different entity with the same name, and the viewer should keep
  watching him. This is the reason the name is stored at all.
- Restoring and advancing are the same scan with different endings, and the caller chooses
  which. Restoring is used whenever a target was lost involuntarily; advancing only on an
  explicit key press.
- The candidate list is built into a fixed-capacity buffer of thirty-two, which silently
  caps how many players can be spectated. A rebuild should size it from the player count.

## `FirstEye_ToPlayer`

**Contract** — moves the level's notion of "the entity being viewed" onto another object,
which is what makes the first-eye mode actually render through somebody else's head. The
order of operations is the whole content:

```text
FUNCTION FirstEye_ToPlayer(target)
  previous <- level.current_entity
  IF previous IS an actor THEN previous.inventory.items_render_first_person(false)
  IF previous IS a spectator THEN re-register it with the scheduler   # see note

  IF target EXISTS THEN
    level.current_entity <- target
    re-register target with the scheduler
    IF target IS an actor THEN target.inventory.items_render_first_person(true)

  IF world is paused AND previous was an actor THEN
    WITH engine frame delta forced to zero: pump previous's client and scheduled updates
```

**Invariants**

- The first-person rendering flag must be cleared on the old target before it is set on the
  new one. Two actors both believing their weapons are the first-person model draw two
  weapons over the camera.
- Re-registering with the scheduler is not a no-op: it removes and re-adds the object so
  that it is updated **first** in the frame. The entity being viewed must have its pose
  finished before the camera reads it, or the view lags the body by a frame.
- The old target is pumped once more if the world is paused, so that it visibly returns to
  its non-first-person appearance instead of freezing mid-transition.

## `cam_Set`

**Contract** — switches the active camera mode, moving the level's view entity as the
transition demands: entering the first-eye mode hands the view to the watched player;
leaving it takes the view back to the spectator itself. The outgoing camera is told it is
deactivating and the incoming one is handed the outgoing one, so it can inherit its
orientation and avoid a snap.

## `net_Spawn`

**Contract** — brings the spectator up from its server record and chooses its opening
camera. Orientation and position come from the record; the yaw and pitch are stored
negated relative to the camera's convention and are inverted on the way in.

```text
FUNCTION net_Spawn(record) -> bool
  IF NOT base.net_Spawn(record) THEN RETURN false
  roll <- 0
  IF not a multiplayer mode, OR the mode permits free flight THEN
    active <- free_fly
  ELSE IF the local player has no team, or is on the spectators' team THEN
    active <- free_fly                 # a declared spectator always gets free flight
  ELSE
    active <- fixed_look_at            # a dead participant is pinned where he fell
    roll   <- -record.angles.roll
  watch_index <- 0
  cameras[active].orient(-record.angles.yaw, -record.angles.pitch, roll)
  cameras[active].position <- record.position
  IF running as the authority THEN record.flags <- locally originated spawn
  RETURN true
```

**Invariants** — the roll is carried into the camera only in the pinned mode. A free-flying
camera must start level; a pinned one inherits the orientation of the death that produced
it, which is how the "you died looking at this" shot works.

## `net_Destroy`

**Contract** — tears the spectator down and tells the map display to drop its marker.
Skipped on a dedicated server, which has no map.

## `net_Relcase`

**Contract** — called when *any* object is about to be destroyed, so that nothing keeps a
reference to it. Only the watched player matters here.

```text
FUNCTION net_Relcase(departing)
  IF departing != watching THEN RETURN
  IF watching IS NOT the level's current entity THEN
    watching <- none ; RETURN          # a new spectator already took over the view
  watching <- none
  IF active != free_fly THEN
    restore_previous_watch_target()
    IF watching = departing THEN watching <- none   # the restore found the same doomed object
  IF watching IS none THEN switch_to(free_fly)
```

**Invariants** — the restored target must be re-checked against the departing object,
because the restore matches by name and the dying player's record still carries that name.
Missing this check leaves a reference to a freed object, which is exactly the failure this
notification exists to prevent.

## `On_SetEntity` / `On_LostEntity`

**Contract** — called when this spectator becomes, and stops being, the view entity. It
saves the global camera-inertia setting, replaces it with a fixed value tuned for
spectating, and restores the saved value on the way out.

**Notes** — the setting is the user's, global and shared, and a spectator overwriting it
means a crash while spectating leaves the player's camera permanently mis-tuned. A rebuild
should make inertia a property of the camera rather than of the process. The fixed value
is a heavier inertia than the default, which smooths the jumpy motion of following someone
else's aim. **Could not recover**: why that particular value.

## `GetSpectatorString`

**Contract** — builds the status line shown to a spectator: the localized word for
"spectator", the localized name of the active mode, and the watched player's name where
one applies. Empty in single player. Every fragment comes from the localization table;
none is a literal.

## `shedule_Update`

**Contract** — nothing beyond the base's, and only once the object is ready. The spectator
does all of its work per frame, not on the scheduler, because its camera must be current
for the frame being rendered.

## `Center` / `Radius`

**Contract** — the spectator's extent for spatial queries: its position, and an
effectively zero radius. It is a point that occupies no space and collides with nothing.
