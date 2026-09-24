# src/Layers/xrRender/SkeletonRigid.cpp

> The pose solve: the two-level rate limit that decides whether to recompute a skeleton at all, the parent-before-child walk that computes every bone, and the periodic rebuild of the model's bounding volume.

**Needs** — [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`Include/xrRender/Kinematics.h`](../../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md) · [`xrRender_console.h`](xrRender_console.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: matrix arithmetic over arrays the file does not own. It stays out of T3 only because it is the hottest non-drawing operation in the frame — it runs for every visible animated model, several hundred times a frame — and it is guarded by a process-wide lock.

## Purpose

This file continues the class declared in [`SkeletonCustom.h`](SkeletonCustom.h.md) and implemented in [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md); it has no type of its own. It holds the part of the *rigid* skeleton that answers "what shape is this model right now": the solve entry point with its rate limiting, the single-bone composition step that external drivers hook into, the recursive walk, and the ancestor-chain walk used for a bone-accurate query that must not disturb the live pose.

The split from its sibling file is arbitrary — one is "the model", the other is "the pose" — and a rebuild should merge them.

## State

`Stateless.` It reads and writes the skeleton's own arrays, and one module-level tunable:

```text
skeleton_box_period : int, default 32     # frames between bounding-volume rebuilds
```

## `CalculateBones`

**Contract** — brings the whole skeleton's pose up to date and, periodically, its bounding volume. Idempotent within one millisecond. Blocks on a **process-wide lock shared by every skeleton**. Allocates nothing. Fires the owner's update notification when it does real work. This is the single most-called expensive operation in a frame.

```text
FUNCTION calculate_bones(force_exact)
  IF now_ms = last_solve_ms
    RETURN                              # the free early-out

  LOCK skeleton_solve_lock DURING
    on_calculate_bones()                # subclass hook: the animated skeleton advances its tracks HERE

    IF NOT force_exact AND now_ms < last_solve_ms + 100 ms
      RETURN                            # the coarse-rate early-out

    IF the sub-mesh partition is dirty
      recompute which sub-meshes have visible bones

    last_solve_ms = now_ms
    solve_subtree(root, identity)
    ASSERT at least one bone is visible

    box_countdown = box_countdown + 1
    IF box_countdown >= skeleton_box_period
      box_countdown = -random_int_below(skeleton_box_period - 1)
      rebuild the bounding box from every visible bone's posed oriented box
      derive the bounding sphere from it

    IF update_callback EXISTS
      update_callback(self)
```

**Invariants** — after it returns, every visible bone's transform is consistent with its parent's and its render transform matches. A skeleton with no visible bones is a fatal inconsistency, not a valid state.

**Notes**

Four timing decisions, and each is the reason a different thing is affordable.

- **The two early-outs are different guards and both are needed.** The first makes a repeated call within one millisecond free, which matters because the renderer, the physics step and several game queries each force a solve on the same model in the same frame. The second is the coarse *rate*: a skeleton nobody has asked about precisely is allowed to be up to **100 milliseconds stale** — ten solves a second. That interval is not a quality setting; it is the answer to "how stale may an off-screen creature's bones be before the AI's own bone queries go wrong", and shipped content is tuned around it. The same constant appears in the inverse-kinematics layer, which uses it to decide whether its cached limb state is still current.

- **The animation hook runs before the coarse early-out, the pose after it.** This ordering is load-bearing: animation *time* must advance at the real frame rate even when the *pose* is only computed ten times a second, or blends would accrue in visible steps and end callbacks would fire late. Invert the two and every animation stutters.

- **The bounding volume is rebuilt every 32 frames, with a randomized phase.** The countdown is reseeded to a *negative random* value rather than to zero, so models that were loaded in the same frame do not all rebuild their boxes on the same later frame. Spreading a periodic cost by randomizing its phase rather than its period is the pattern worth keeping. The consequence is that a model's culling volume lags its pose by up to half a second — acceptable because the box is derived from bone boxes that are generous to begin with.

- **The lock is global, not per skeleton.** Every skeleton in the process serializes on one lock. That is a heavy decision and it is not explained anywhere; the plausible reason is that solves can be entered from the loading thread and from game code at the same time and the cheapest correct answer was one lock. A rebuild that solves skeletons in parallel must replace it with per-instance exclusion and audit the update callbacks, which run inside it and reach into game state.

## `CLBone` — one bone, and where external drivers get their turn

**Contract** — compute one bone instance's transform given its parent's, honouring the two kinds of external driver, and derive the render transform. Does nothing for a hidden bone — which is what makes a hidden bone's whole subtree skip too, since the walk still descends but every descendant is also skipped only if it is itself hidden.

```text
FUNCTION solve_bone(bone, instance, parent_transform, channel_mask)
  IF NOT visible(bone): RETURN

  IF instance.callback_overwrite
    IF instance.callback EXISTS
      instance.callback(instance)        # the driver supplies the pose outright; no animation is sampled
  ELSE
    build_bone_matrix(bone, instance, parent_transform, channel_mask)
    ASSERT the result is a finite matrix          # "animation killed the bone matrix"
    IF instance.callback EXISTS
      instance.callback(instance)        # the driver corrects the pose in place
      ASSERT the result is still finite           # "the callback killed the bone matrix"

  instance.render_transform = instance.transform composed with bone.model_to_bone
```

**Notes** — The two assertions are not defensive clutter; they are how the engine localizes a class of bug that is otherwise invisible until a model explodes across the level. One blames the animation data, the other blames whichever subsystem installed the driver. They are compiled out of the shipping build, and a rebuild should keep them in development builds for the same reason.

The overwrite flag is a real optimization and not just a policy: when physics owns a bone, sampling and blending its animation would be wasted work, so the whole build step is skipped.

## `BuildBoneMatrix` — the rigid pose

**Contract** — the rigid skeleton's answer to "what is this bone's local pose": the bind transform, composed onto the parent, plus any script-authored offsets. The animated skeleton overrides this to sample blends instead.

```text
FUNCTION build_bone_matrix(bone, instance, parent, channel_mask)
  instance.transform = parent composed with bone.bind_transform
  apply_bone_offsets(bone, instance, parent, channel_mask)
```

**Notes** — That a rigid skeleton poses every bone at its bind transform means a rigid skeleton is *rest-posed unless something drives it*. Every rigid skeleton in the game — doors, vehicles, ragdolls, destructibles — is therefore driven entirely through per-bone callbacks. The default is not a fallback, it is the base case.

## `CalculateBonesAdditionalTransforms` — script-authored offsets

**Contract** — apply every registered offset whose target is this bone, in registration order, after the pose is built. Scans the offset list per bone per solve; the list is almost always empty.

```text
FUNCTION apply_bone_offsets(bone, instance, ...)
  FOR EACH offset WHERE offset.bone_id = bone.id
    saved_position   = instance.transform.translation
    instance.transform = instance.transform composed with offset.transform  # rotation, in bone axes
    instance.transform.translation = saved_position + offset.translation    # translation, in the parent frame
```

**Notes** — The rotation and the translation of one offset are applied in **different spaces**, deliberately: the rotation composes in the bone's own axes (so "rotate this hand five degrees" means five degrees about the hand's axes), while the translation is added in the frame the bone's position already lives in (so "move it two centimetres up" means up in model space, not along whatever axis the bone happens to point). Composing both in one space would make either half unusable for the thing it exists for. Preserve the split.

## `LL_AddTransformToBone` · `LL_ClearAdditionalTransform`

**Contract** — append an offset; remove every offset for one bone, or all of them. The sentinel bone id means "all". Linear.

## `Bone_Calculate` — the walk

**Contract** — solve a subtree, parent first. Called on the root with the identity transform.

```text
FUNCTION solve_subtree(bone, parent_transform)
  solve_bone(bone, instance_of(bone), parent_transform, all channels)
  FOR EACH child IN bone.children
    solve_subtree(child, instance_of(bone).transform)
```

**Invariants** — a parent is always computed before its children, which is the entire correctness requirement of the pose system. The hierarchy walk guarantees it structurally; the array order also happens to satisfy it, which other parts of the engine rely on.

**Notes** — The full solve passes an **all-channels mask**, not the default single-channel one. Every other entry point defaults to channel zero alone; only the real solve mixes every animation channel. A rebuild that lets the default leak into the solve silently loses every additive layer — aim offsets, lean, recoil.

## `BoneChain_Calculate` and `Bone_GetAnimPos` — asking without disturbing

**Contract** — compute one bone's transform by walking *up* to the root and back down that chain alone, using copies of the bone instances rather than the live ones. Optionally suppress the external drivers so the caller sees the pose animation alone would produce. Leaves the skeleton untouched.

```text
FUNCTION chain_pose(bone, out instance_copy, channel_mask, ignore_drivers)
  saved_driver = instance_copy's callback and overwrite flag
  IF ignore_drivers
    clear the callback on the copy
  IF bone is the root
    solve_bone(bone, instance_copy, identity, channel_mask)
  ELSE
    parent_copy = a COPY of the parent's live instance
    chain_pose(parent, parent_copy, channel_mask, ignore_drivers)
    solve_bone(bone, instance_copy, parent_copy.transform, channel_mask)
  restore the saved driver on the copy
```

**Notes**

- This is how a ragdoll learns what pose the animation *would* have produced, so it can blend back towards it; and how the game asks where a bone will be without committing to it.
- Copying each ancestor's instance is what makes the walk non-destructive, and it is also what makes the cost proportional to the chain depth rather than to the bone count — the expensive alternative would be solving the whole skeleton twice.
- Saving and restoring the driver on a *copy* is redundant work, since the copy is discarded; it is the shape of a function that was once written against the live instance. Harmless, and a rebuild drops it.

## `check_kinematics`

**Contract** — debug builds only: if the root bone has been flung far outside the world, dump every bone's matrix and fail. Called after every full solve.

**Notes** — It tests one number against a fixed altitude threshold, which is crude, but it catches the failure mode that actually happens: a corrupt blend or a misbehaving driver produces one bad matrix, the error propagates down the hierarchy, and the model ends up somewhere absurd. Catching it at the frame it happens, with the whole hierarchy dumped, is the difference between a five-minute diagnosis and a week of one. The separate check on the bounding sphere's radius inside the volume rebuild is the same guard from the other end.
