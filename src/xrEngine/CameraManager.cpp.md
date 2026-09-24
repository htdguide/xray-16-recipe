# src/xrEngine/CameraManager.cpp

> Runs the two effector stacks over the frame's camera description and writes the result into the device as a view matrix, a projection matrix and a post-process parameter set.

**Needs** — [`CameraManager.h`](CameraManager.h.md) · [`CameraBase.h`](CameraBase.h.md) · [`Effector.h`](Effector.h.md) · [`EffectorPP.h`](EffectorPP.h.md) · [`Environment.h`](Environment.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`device.h`](device.h.md) · [`Render.h`](Render.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`CameraManager.h`](CameraManager.h.md); callers name that, not this file.
**Tier floor** — T2: matrix construction and list surgery. It is above the device only because it hands the finished matrices across the renderer interface rather than to a driver.

## Purpose

Exactly one place per frame decides where the camera is. The active camera proposes a basis; this smooths it, lets a stack of effectors perturb it, composites a second stack of screen effects, and commits the result. Everything downstream — visibility, culling, the audio listener, the sky — reads the committed values, so the commit point is the frame's spine.

The split between "propose" and "commit" exists because the proposal is untrustworthy: an effector may push the camera through a wall, may want the frame skipped entirely, and may invalidate the orthogonality of the basis. The manager owns fixing all three.

## State

```text
RECORD CameraManager
  cam_info          : CameraInfo         # the frame's camera; survives across frames (it is smoothed)
  cam_effectors     : list<CamEffector>  # ordered; front runs LAST — see apply order
  cam_effectors_new : list<CamEffector>  # staged additions, merged after the stack runs
  pp_effectors      : list<PPEffector>
  auto_apply        : bool               # commit inside update, or let the caller commit
  pp_affected       : PostProcessParams  # the composited screen-effect state

  # invariant: at most one live effector per identity, in each stack.
  # invariant: cam_info.{d,n,r} are orthonormal on exit from update.
```

Two module-wide post-process constants are defined here and are the algebraic basis of the compositing rule below:

- **identity** — the no-op parameter set. Note this is *not* all zeros: the colour-grading base is a mid-grey triple and the grey-conversion weights are a third each, because those parameters are multiplicative offsets around a neutral point, not additive intensities. Film grain size is 1, grain rate 30 per second.
- **zero** — genuinely all zeros, including grain size and rate. It is the accumulator's starting value, never a state anything is left in.

Two module-wide camera constants control smoothing: a positional/rotational inertia, and a separate slide inertia used by the game's lean and strafe code. The positional one ships at zero — the camera is rigid by default and the game raises it situationally.

## `update`

**Contract** — Advances the camera one frame from a proposed basis, a target field of view, a target aspect and a target far plane, plus the proposing camera's rigidity flags. Runs both effector stacks, optionally commits to the device, and merges staged effector additions. Must be called exactly once per frame while the simulation is running; calling it twice in one frame silently doubles the smoothing and is checked for in debug builds. Does not allocate in the common case.

```text
FUNCTION update(p, d, n, fov_target, aspect_target, far_target, flags)
  # 1. Adopt the proposal, smoothing the axes the camera did not mark rigid.
  IF flags has position_rigid  THEN cam_info.p = p  ELSE cam_info.p = lerp_toward(cam_info.p, p, cam_inert)
  IF flags has direction_rigid THEN cam_info.d = d ; cam_info.n = n
  ELSE
    cam_info.d = lerp_toward(cam_info.d, d, cam_inert)
    cam_info.n = lerp_toward(cam_info.n, n, cam_inert)
  orthonormalize(cam_info)

  # 2. Smooth the lens toward its target at a rate tied to real elapsed time.
  #    10 per second: a field-of-view or far-plane change settles in about a
  #    tenth of a second regardless of frame rate. Clamped so a long frame
  #    snaps rather than overshooting.
  blend = clamp(10 * frame_delta_seconds, 0, 1)
  cam_info.fov    = mix(cam_info.fov,    fov_target,               blend)
  cam_info.far    = mix(cam_info.far,    far_target,               blend)
  cam_info.aspect = mix(cam_info.aspect, aspect_target * device_aspect, blend)
  cam_info.near   = engine_near_plane          # never smoothed, never overridden
  cam_info.dont_apply = false

  run_cam_effectors()
  run_pp_effectors()

  IF NOT cam_info.dont_apply AND auto_apply THEN apply_to_device()
  merge_staged_effectors()
```

**Invariants** — On return the basis is orthonormal and the near plane is the engine's constant. The far plane comes from the weather system's current far plane, not from the camera, so fog distance and clip distance can never disagree.

**Notes** — `aspect_target` is multiplied by the device's own height-over-width ratio rather than replacing it. The camera's aspect field is therefore a *correction factor* applied on top of the window's real shape — a camera authored for a wide letterboxed cutscene sets it to something other than one, and a normal camera sets one and inherits the window.

## `run_cam_effectors`

**Contract** — Runs every live camera effector over the frame's camera description, newest first, and drops the ones that report themselves finished. Re-orthogonalizes afterwards because effectors are allowed to leave the basis skewed.

```text
FUNCTION run_cam_effectors()
  IF stack is empty THEN RETURN
  FOR EACH effector IN stack, from BACK to FRONT
    IF process(effector) THEN keep it
    ELSE
      fire its removal callback
      remove and destroy it
  orthonormalize(cam_info)

FUNCTION process(effector) -> bool     # "keep me"
  IF effector.is_valid AND effector.apply(cam_info) THEN RETURN true
  IF effector.wants_invalid_processing THEN effector.apply_expired(cam_info)
  RETURN false
```

**Notes** — The back-to-front order is what makes the insertion rule meaningful: an effector that positions the camera absolutely is pushed to the **front** of the list and therefore runs **last**, overriding everything that merely perturbs. Relative effectors are appended to the back and run first. This is the whole ordering policy, and it is stated nowhere in the data.

**Notes** — An effector is never destroyed inside its own `apply`. The original spells out why: the stack is being walked, and a subclass that overrode the per-effector step would be handed a pointer the base had already freed. The step therefore *reports* that an effector is finished and the walk does the removal. A rebuild with ownership in the container gets this free, but must still not allow the callback to re-enter the stack — the removal callback runs during the walk.

**Notes** — `apply_expired` is the escape hatch for an effector that must do something on its last frame even though it has run out of time — restoring a field of view it pushed, for example. Without it, an effector's final frame would be silently dropped.

## `run_pp_effectors`

**Contract** — Composites the post-process effector stack into one parameter set. Two composition modes coexist: additive effects accumulate as deviations from the identity set, and a non-overlapping effect *replaces* the accumulation outright and stops anything below it from contributing. Drops effectors that report themselves finished.

```text
FUNCTION run_pp_effectors()
  IF stack is empty THEN pp_affected = identity ; RETURN

  replaced = false
  accum    = identity
  live     = 0
  FOR EACH effector IN stack, from BACK to FRONT
    contribution = zero
    IF NOT (effector.is_valid AND effector.apply(contribution)) THEN
      remove effector ; CONTINUE
    live = live + 1
    IF NOT replaced THEN
      accum = accum + contribution - identity   # add the deviation, not the value
    IF effector.is_exclusive THEN
      replaced = true
      accum = contribution                      # this effect owns the screen
  IF live == 0 THEN accum = identity ELSE normalize(accum)

  IF accum.grain is not positive THEN accum.grain = identity.grain
  pp_affected = accum
```

**Notes** — `accum + contribution - identity` is the reason the identity set is not zeros. Each effector reports an absolute parameter set; subtracting the identity turns it into a deviation so that two half-strength effects sum to one full-strength one rather than to twice the neutral value. `normalize` then re-clamps the colour-grading fields into their legal range.

**Notes** — The grain-size guard is a floor, not a clamp: a composited grain size of zero would divide by zero in the shader, so it falls back to the neutral value. The parameter is separately clamped away from zero again at commit time, which is belt and braces.

**Notes** — The exclusive effector still contributes to `live`, so a single exclusive effect does not collapse to identity. Because the walk is back-to-front and exclusivity latches, the *front-most* exclusive effector wins.

## `apply_to_device`

**Contract** — Commits the frame's camera. Builds the view matrix from the basis, publishes the basis and the lens parameters where the rest of the engine reads them, builds the projection matrix, and hands the composited post-process parameters to the renderer. When the main menu is up, the post-process state is reset to neutral instead, so a screen effect running when the player opens the menu does not bleed into it.

```text
FUNCTION apply_to_device()
  device.view    = look_along(cam_info.p, cam_info.d, cam_info.n)
  device.camera_{position, direction, top, right} = cam_info.{p, d, n, r}
  device.fov     = cam_info.fov
  device.aspect  = cam_info.aspect
  device.project = perspective(radians(cam_info.fov), cam_info.aspect, cam_info.near, cam_info.far)
  shear device.project by (-offset_x, -offset_y)

  IF main menu is active THEN reset_post_process()
  ELSE
    clamp pp_affected.grain to (epsilon, 1000)
    renderer.set_post_process(pp_affected)
```

**Notes** — The projection shear is two matrix entries an external screenshot tool drives to render a scene as a grid of offset tiles and stitch them into a very high resolution image. It is zero in normal play; the entries are written unconditionally rather than guarded, which is cheaper than a branch and equally correct.

## `reset_post_process`

**Contract** — Publishes the neutral post-process set with colour-grading influence at zero, interpolation complete, and no grading textures bound. Distinct from "the identity set" because the grading texture slots must be *cleared*, not merely neutral — leaving a texture bound at zero influence keeps it resident.

## `add_cam_effector`

**Contract** — Stages an effector for insertion; it is not live until the current frame's stack walk has finished. Ownership transfers to the manager. Returns the effector so the caller can keep configuring it.

**Notes** — Staging is not an optimization. Game code adds effectors from inside effector callbacks and from inside the frame update, which is exactly when the list is being walked. Deferring the insertion to a single merge point after the walk is what makes that safe. The merge point also performs the one-per-identity removal, so an effector that replaces another does not disturb the walk either.

## `remove_cam_effector` / `remove_pp_effector`

**Contract** — Removes the live effector with the given identity, fires its removal callback and destroys it. A post-process effector may opt out of being destroyed on removal, for the case where the game owns it and reuses it. Removing an identity that is not present is a no-op, not an error — the game removes optimistically.

## `get_cam_effector` / `get_pp_effector`

**Contract** — Finds the live effector with an identity, or reports absence. Linear scan; the stacks are a handful of entries deep in practice.

## `request_cam_effector_id` / `request_pp_effector_id`

**Contract** — Allocates an unused identity for a game-defined effect, by scanning upward from a fixed threshold until an unoccupied identity is found. The threshold is above every engine-reserved identity; its exact value is arbitrary.

**Notes** — An allocated identity is only unique while the effector using it is alive, since the scan reuses gaps. That is intended — the identity is a slot, not a handle — but it means a caller must not cache an allocated identity past its effector's lifetime.

## `update_from_camera`

**Contract** — The normal entry point: takes a camera, reads its basis, lens and flags, takes the far plane from the weather system's current environment, and runs `update`.

## `dump`

**Contract** — Logs the committed camera basis, recovered by inverting the view matrix. Debug only, and it reads the *device's* matrix rather than the manager's own record deliberately — it is used to check that the commit did what the manager intended.
