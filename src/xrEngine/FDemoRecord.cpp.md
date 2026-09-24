# src/xrEngine/FDemoRecord.cpp

> A free-flying camera that records its own keyframes, and the three screenshot modes — plain, cube map, and the orthographic level map — built on top of it.

**Needs** — [`FDemoRecord.h`](FDemoRecord.h.md) · [`Effector.h`](Effector.h.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`xr_level_controller.h`](xr_level_controller.h.md) · [`xr_input.h`](xr_input.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`Environment.h`](Environment.h.md) · [`CustomHUD.h`](CustomHUD.h.md) · [`GameFont.h`](GameFont.h.md) · [`Render.h`](Render.h.md) · [`CameraManager.h`](CameraManager.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: camera integration and state sequencing. The screenshot modes reach the graphics device but only through the renderer interface.

## Purpose

This is the engine's content-authoring camera. It does four separable jobs and they are together because they all need a camera detached from the player and a way to suppress the heads-up display:

1. **Record a demo** — fly around and stamp keyframes, producing the path [`FDemoPlay.cpp`](FDemoPlay.cpp.md) replays.
2. **Take a screenshot** with the interface hidden.
3. **Capture a cube map** — six faces from one point, which is how the game's environment reflection probes are authored.
4. **Capture the level map** — a top-down orthographic render of the whole level, which becomes the in-game map texture.

It is simultaneously a camera effector, an input receiver and a render handler. Those three roles are the whole design: it overrides the camera, it takes the keyboard away from the game, and it draws its own help overlay.

## State

```text
RECORD DemoRecorder
  file              : writer                 # keyframes stream out as they are stamped
  keyframe_count    : int
  camera            : matrix4                # the flying camera's transform
  position          : vector3
  hpb               : vector3 (radians)      # heading, pitch, bank
  translate_request : vector3                # accumulated this frame from input
  rotate_request    : vector3
  velocity          : vector3                # smoothed translate request
  angular_velocity  : vector3                # smoothed rotate request

  speed_step        : one of four            # four authored translation speeds
  angle_speed_step  : one of four            # four authored rotation speeds
  speeds[4], angle_speeds[4] : real          # read from configuration

  mode              : none | screenshot | cubemap | levelmap
  stage             : int                    # step counter within a multi-frame mode
  levelmap_quadrant : int                    # -1 for one shot, 0..3 for the four-tile capture
  saved_weather     : text                   # the cycle to restore after a level-map capture
  redirect_to_level : bool                   # pass input through to the game instead
  help_font         : font
  global_position   : shared vector3         # see Notes
```

**Invariants** — The recorder owns the keyboard while active, and everything it does to global state — the heads-up display flags, the device flags, the window style, the weather cycle, the suppression of on-screen error text — is saved before the change and restored after. A mode that exits without restoring leaves the game visibly broken.

## `open`

**Contract** — Opens the output file, deleting any existing one, captures input, and derives the flying camera's starting orientation from wherever the game's camera currently is so that recording begins from the player's view. Reads the eight speed values from configuration. If the file cannot be opened, the recorder expires immediately instead of failing.

```text
FUNCTION open(name)
  register as a render handler at low priority minus 1000   # draws over everything
  suppress on-screen error text                              # it would appear in recordings
  delete any existing file at name ; file = open_for_write(name)
  IF file is absent THEN lifetime = -1 ; RETURN              # expire on the next stack walk

  capture input
  camera = inverse of the device's current view matrix
  # Recover heading from the forward vector's horizontal projection, and pitch
  # from its vertical component. Bank starts at zero: the game's camera has none.
  heading = angle of the forward vector in the horizontal plane, in the full circle
  pitch   = arcsin(forward.vertical)
  bank    = 0
  position = camera's translation
  read speeds and angular speeds from the "demo_record" configuration section
```

**Notes** — Heading is recovered with a two-branch arccosine rather than a two-argument arctangent, producing a value in the full circle. The branch is on the sign of the horizontal *right* component, which is the standard way to disambiguate an arccosine and is equivalent to the two-argument form.

## `apply`

**Contract** — The per-frame step, called as a camera effector. Dispatches on the active mode: a screenshot, cube map or level-map capture runs its multi-frame sequence and the camera does what that sequence needs; otherwise the free-flying camera integrates this frame's input and writes its basis into the frame's camera description. Draws the help overlay while a help key is held. Never reports expiry — the recorder ends by having its lifetime set negative from a key press.

```text
FUNCTION apply(cam_info) -> bool
  IF no file THEN RETURN alive
  IF mode is screenshot  THEN run screenshot step  ; camera from the flying camera
  ELSE IF mode is levelmap THEN run level-map step ; cam_info.suppress_device_write = true
  ELSE IF mode is cubemap  THEN run cube-map step  ; aspect = 1
  ELSE
    IF help key held THEN draw the key legend
    # Smooth the raw input requests toward the camera. 0.3 per frame is the feel:
    # sharp enough to respond, damped enough that a recorded path is not jittery.
    velocity         = lerp(velocity,         translate_request, 0.3)
    angular_velocity = lerp(angular_velocity, rotate_request,    0.3)
    step        = velocity         * frame_delta * speeds[speed_step]
    angle_step  = angular_velocity * frame_delta * angle_speeds[angle_speed_step]
    heading = heading - angle_step.y ; pitch = pitch - angle_step.x ; bank = bank + angle_step.z
    IF an external position was pushed THEN adopt it ELSE publish the current one
    position = position + camera.forward*step.z + camera.right*step.x + camera.up*step.y
    camera = transform from (heading, pitch, bank) translated to position
    write the camera's basis into cam_info
    lifetime = lifetime - frame_delta
    clear translate_request and rotate_request
  RETURN alive
```

**Notes** — The input requests are cleared at the end of every frame and re-accumulated from held-key callbacks during the next. That makes the recorder's motion depend on the input layer delivering a *hold* event per frame per held key, which is exactly what [`xr_input.cpp`](xr_input.cpp.md) guarantees.

**Notes** — Translation uses the camera's own axes, so movement is always relative to where the camera looks, including vertically — "up" is the camera's up, not the world's. For a flying authoring camera that is the right choice and it is why a banked camera moves sideways when asked to rise.

**Notes** — The shared global position is a static slot the game can write to teleport the recorder, and which the recorder otherwise keeps filled with its own position so the game can read where it is. It is a one-value channel between two subsystems that have no other connection; a rebuild passes it explicitly.

## `record_keyframe`

**Contract** — Writes the *inverse* of the camera transform — that is, the view matrix — to the file and increments the count. One transform, no header, appended. This is the whole file format, and the reason the reader validates only that the length is a multiple of a transform.

## `capture_screenshot`

**Contract** — A two-frame sequence: on the first frame save and clear the heads-up display flags, on the second take the shot and restore them. Two frames, not one, because the flags are read during the render pass that has already been set up for the current frame — clearing them takes effect next frame.

## `capture_cubemap`

**Contract** — A seven-frame sequence that points the camera along each of the six axes in turn, takes a shot after each, and restores the original orientation and the display flags at the end. The face order and the up vector for each face are fixed: +X, −X, +Y, −Y, +Z, −Z, with the world up used for the four horizontal faces and the forward and backward axes used as up for the two vertical ones.

**Notes** — Frame zero only *aims* at the first face; the shot for face zero is taken on frame one, after the render pass has actually used that orientation. Hence seven frames for six faces. The same one-frame lag is why the screenshot mode takes two.

**Notes** — The face order and up vectors must match whatever the texture pipeline expects for a cube map; they are not free. This is the authoring side of the reflection probes the renderer samples.

## `capture_level_map`

**Contract** — Renders the whole level from directly above under an orthographic projection, producing the texture the in-game map uses. Optionally in four quadrants at full resolution each, for a high-quality map. Forces exclusive fullscreen, strips the renderer down to static geometry on a cleared buffer, switches the weather to a dedicated flat-lighting cycle, waits out the device reset and a further settling period, captures, then restores every one of those.

```text
FUNCTION begin_level_map(high_quality)
  saved_weather = current weather cycle
  set weather to the cycle named "map", forced      # flat, shadowless lighting
  quadrant = IF high_quality THEN 0 ELSE -1
  bounds = the level's authored map rectangle, or its bounding volume if it has none
  IF quadrant >= 0 THEN narrow bounds to that quadrant
  mode = levelmap ; stage = 0

FUNCTION level_map_step()
  IF stage == 0 THEN
    save display flags, device flags and window style
    device flags = {clear buffer, draw static geometry} only
    force exclusive fullscreen ; reset the device if the style changed
  ELSE IF stage == reset_precache_frames + 30 THEN
    build the top-down orthographic camera over the current bounds
    capture, naming the file after the level and, in quadrant mode, the quadrant index
    IF quadrant mode AND more quadrants remain THEN
      advance to the next quadrant, re-derive the bounds, rewind stage by 20
    ELSE restore flags, window style, weather; reset the device if needed
  ELSE
    rebuild the orthographic camera            # every frame, so the reset does not lose it
  stage = stage + 1
```

**Notes** — The wait is the device's precache frame count plus thirty. The precache part is mandatory — the device has just been reset into fullscreen and its resources are being rebuilt — and the extra thirty frames let streamed textures and the weather change settle. Capturing early produces a map with missing textures, which is exactly the failure this constant exists to prevent.

**Notes** — Rewinding the stage by twenty rather than resetting it to zero is what makes the quadrant loop skip the device-reset step and re-enter the settling window with twenty frames to go. It is a state machine expressed as arithmetic on a counter; a rebuild should make the phases explicit.

**Notes** — The level's map bounds come from an authored rectangle in the level's own configuration when one exists, and from the level's bounding volume otherwise. The authored rectangle is what makes the in-game map line up with world coordinates, and it is in the level data, so a rebuild must read it.

**Notes** — The dedicated `map` weather cycle exists purely so the capture has no sun, no shadows and no fog. It ships with the game data.

## Input handling

**Contract** — While active, the recorder receives every key, mouse and gamepad event before the game does, and may pass them through instead. One key toggles that pass-through, so an author can drive the player around and then take the camera back.

The bindings split in two:

- **By action**, through the engine's binding table, so the recorder follows whatever the player has bound: the movement actions drive translation, the two lookout actions drive yaw, the accelerate/sprint/crouch actions select a speed step, the fire and zoom actions move forward and backward, and the console, screenshot, quit, pause and editor actions do what they say. Jump stamps a keyframe.
- **By raw scancode**, for keys the game does not bind: the numeric keypad drives strafing, pitch, yaw and bank directly; a function key captures a cube map; another captures the level map, in high quality when a modifier is held; a further key shows the help legend.

**Notes** — Speed selection is duplicated: the bound actions *and* the raw modifier scancodes both select a speed step. The raw path is the fallback for a player who has rebound those actions away, which would otherwise leave the recorder with no speed control. Releasing any of them returns to the middle speed rather than to the previous one.

**Notes** — Every input contribution is divided by the game's time factor before being accumulated. The motion is then multiplied by the *scaled* frame delta in the per-frame step, so the two cancel and the recorder flies at a constant real-world speed regardless of time acceleration. Without this, recording during bullet time would crawl.

**Notes** — Gamepad sticks select a speed step from the stick's deflection magnitude — four bands, at roughly 0.45, 0.75 and 0.9 — rather than scaling the movement continuously. That gives an analogue stick discrete speeds, which makes a recorded path easier to keep steady than a continuous mapping would. The stick's own axis values are then dead-zoned at 0.35 for directional movement.

**Notes** — Vertical mouse look is scaled by three quarters relative to horizontal, the standard compensation for a screen being wider than it is tall.

## `on_render`

**Contract** — Draws the accumulated help text. Registered at a very low render priority so it is drawn last, over everything.

**Notes** — The help font is chosen by matching the window *height* against three authored texture sizes — up to 600, up to 1024, above — and falling back to progressively smaller ones if the chosen size is not present in the font's configuration section. An equivalent width-based selection is present and disabled. The decision is that a bitmap font needs a texture sized for the display, and height is the better predictor because interface layout is vertical.
