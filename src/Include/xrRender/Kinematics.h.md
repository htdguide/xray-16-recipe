# src/Include/xrRender/Kinematics.h

> The skeleton interface: everything the game, physics and AI layers are allowed to know about an animated model's bones.

**Needs** — [`RenderVisual.h`](RenderVisual.h.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md) · [`Layers/xrRender/KinematicsAddBoneTransform.hpp`](../../Layers/xrRender/KinematicsAddBoneTransform.hpp.md) · [`KinematicsAnimated.h`](KinematicsAnimated.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`KinematicsAnimated.h`](KinematicsAnimated.h.md) · [`RenderVisual.h`](RenderVisual.h.md) · [`FSkinnedTypes.h`](../../Layers/xrRender/FSkinnedTypes.h.md) · [`SkeletonCustom.cpp`](../../Layers/xrRender/SkeletonCustom.cpp.md) · [`SkeletonCustom.h`](../../Layers/xrRender/SkeletonCustom.h.md) · [`SkeletonRigid.cpp`](../../Layers/xrRender/SkeletonRigid.cpp.md) · [`SkeletonMotions.cpp`](../../xrCore/Animation/SkeletonMotions.cpp.md) · [`cf_dynamic_mesh.cpp`](../../xrEngine/cf_dynamic_mesh.cpp.md) · [`xr_collide_form.cpp`](../../xrEngine/xr_collide_form.cpp.md) · [`xr_efflensflare.cpp`](../../xrEngine/xr_efflensflare.cpp.md) · [`ActorAnimation.cpp`](../../xrGame/ActorAnimation.cpp.md) · [`ActorHelmet.cpp`](../../xrGame/ActorHelmet.cpp.md) · [`ActorVehicle.cpp`](../../xrGame/ActorVehicle.cpp.md) · [`Actor_Network.cpp`](../../xrGame/Actor_Network.cpp.md) · _and 92 more_
**Tier floor** — T2: a pure interface, but it hands out *aliased mutable references* into the live bone array so that physics and inverse kinematics can overwrite a bone's transform between the animation solve and the skinning read; a tier that copies aggregates on return breaks that contract and must replace it with explicit write-back.

## Purpose

This is the widest and most consequential interface in the whole renderer boundary. A *skeleton* — a hierarchy of named bones, each with a bind transform, an oriented bounding box, a mass and a material — lives inside the renderer because the renderer owns the mesh the bones deform. But almost everything that *drives* a skeleton lives outside it: the physics layer writes ragdoll poses into bones, inverse kinematics adjusts feet, the weapon layer reads muzzle and grip bone positions, the damage layer maps a hit to a bone to pick a hit zone and a material, and scripts attach objects to bones by name. This header is the treaty that lets all of that happen without any of those layers knowing what a mesh is.

It is separate from [`KinematicsAnimated.h`](KinematicsAnimated.h.md) on a real axis: a skeleton that *poses* (a door, a vehicle, a ragdoll — bones driven entirely from outside) is a strictly smaller thing than a skeleton that *plays animations*. Many models are the first and never the second, and the game asks which it got by a downcast query.

## State

The interface owns no state of its own — an implementor does. But the interface pins down what that state must look like, because callers reach into it by index and hold references across calls.

```text
RECORD Skeleton                    # what an implementor must have
  bones          : list<BoneData>          # static, shared between every instance of this model
  instances      : list<BoneInstance>      # per-instance, one per bone, parallel to `bones`
  root           : int                     # index of the bone Bone_Calculate starts from
  visible_mask   : int (64-bit)            # invariant: bit b set => bone b participates
  name_index     : map<text,int>           # bone name -> index, built once at load
  user_data      : optional<Config>        # the model's own ltx block, parsed at load
  extra_offsets  : list<BoneOffset>        # script-authored constant corrections
  bind_box       : Box                     # union of every bone box in bind pose
  update_stamp   : int                     # wall-clock ms of the last full solve
  box_countdown  : int                     # frames until the visibility box is rebuilt

RECORD BoneData                    # static; see xrCore/Animation/Bone.hpp
  self_id, parent_id : int
  children           : list<int>
  bind_transform     : Matrix              # parent-relative rest pose
  model_to_bone      : Matrix              # invariant: inverse of the bone's bind pose in model space
  obb                : Box (oriented)
  shape, ik_limits, mass, center_of_mass, material : ...

RECORD BoneInstance                # per-instance, mutable, aliased by callers
  transform          : Matrix              # bone -> model space, result of the solve
  render_transform   : Matrix              # invariant: transform . model_to_bone; what skinning reads
  callback           : optional<fn>        # driver installed by physics / IK / script
  callback_overwrite : bool                # true => the driver supplies the pose, skip animation
  params             : list<real>[4]       # four scalars a driver and a shader can share

RECORD BoneOffset
  bone_id       : int
  transform     : Matrix
  rotate_global : bool     # axes fixed in model space, or carried with the bone
```

**The invariant that is written nowhere else**: `render_transform` is not an independent value. Skinning multiplies a vertex, already expressed in model space in bind pose, by `render_transform`; therefore `render_transform` must be recomputed from `transform` in the *same* pass that produced `transform`, and any external driver that overwrites `transform` must let the solve recompute `render_transform` afterwards — never write one without the other.

## `IKinematics` — what an implementor must provide

**Contract** — an implementor is a *model instance*: it shares the immutable bone hierarchy with every other instance of the same model, and owns its own array of bone instances. It must answer every query below without allocating, because most of them are called several times per bone per frame from the game layer. None of it is thread-safe; the engine serializes skeleton solves behind a single lock shared by all skeletons (see `CalculateBones`).

**Invariants**

- Bone indices are dense, start at zero, and are stable for the lifetime of the instance. Index `none` is the all-ones 16-bit value and means "no bone" everywhere in the engine.
- A parent always has a lower index than its children, so a single forward pass over the array is a valid topological order — the recursive solve relies on this only for its assertions, but the physics layer relies on it for real.
- At least one bone is always visible. The solve asserts this; a model with an all-zero visibility mask is a bug, not a degenerate case.

### Naming and lookup

```text
FUNCTION bone_id(name : text) -> int              # `none` if absent
FUNCTION bone_count() -> int
FUNCTION visible_bone_count() -> int
FUNCTION bone_name(id) -> text                    # debug builds only
FUNCTION bones() -> list<(text, int)>             # the whole name->index table, sorted
FUNCTION user_data() -> optional<Config>          # the model's ltx block
```

**Notes** — `user_data` is how *authored data travels with the model*. An artist puts an `ltx` fragment in the model file; the game reads it to learn that this model has a headlight, which bone the hit zones hang off, how a door swings. The renderer neither parses nor understands it; it only carries it. A rebuild must keep this channel, because a large amount of shipped content configures itself through it.

### The bone arrays

```text
FUNCTION bone_instance(id) -> ref BoneInstance    # mutable, aliased, valid until the model is freed
FUNCTION bone_static_data(id) -> ref BoneData     # shared; mutating it affects every instance
FUNCTION bone_info(id) -> BoneData (read-only)    # the narrow, read-only view physics uses
FUNCTION transform(id) -> ref Matrix              # == bone_instance(id).transform
FUNCTION render_transform(id) -> ref Matrix       # == bone_instance(id).render_transform
FUNCTION bone_box(id) -> ref Box                  # the bone's oriented box, in bone space
FUNCTION model_box() -> Box                       # bind-pose bounds of the whole model
FUNCTION bind_transforms() -> list<Matrix>        # every bone's bind pose, model-relative
FUNCTION bone_groups() -> list<list<int>>         # bones grouped by the sub-mesh they skin
```

**Notes** — Two access paths to the same bone exist and the split is load-bearing. The wide mutable handle (`bone_instance`) is for drivers that intend to *write* a pose; the narrow read-only view (`bone_info`) is what the physics and IK layers take, and it deliberately exposes only geometry, mass, material and joint limits — enough to build a rigid body from a bone, and nothing that would let physics reach back into the renderer. A rebuild should keep these two views distinct even though one type could serve both; the narrowing is the only thing preventing the physics module from depending on the renderer's model representation.

`bone_groups` reports which bones influence which sub-mesh. It exists because the renderer splits a skinned model into draw-call-sized pieces whose bone sets fit one constant-buffer palette, and the game occasionally needs to know the grouping to hide a body part.

### Visibility

```text
FUNCTION bone_visible(id) -> bool
FUNCTION set_bone_visible(id, visible, recursive)
FUNCTION visible_mask() -> int (64-bit)
FUNCTION set_visible_mask(mask : int (64-bit))
```

**Contract** — hiding a bone hides the geometry weighted to it and removes it from the solve and from the model's bounding box. `recursive` applies the change to the whole subtree; without it, a hidden bone's children keep their own flags, and a child of a hidden bone is simply not reached by the solve, so its transform goes stale.

**Notes** — The mask is 64 bits, which caps a model at **64 bones for visibility purposes** even though bone indices are 16-bit. This is a real, undocumented limit that shipped content lives within: it is how the game toggles a corpse's severed limbs, an NPC's backpack or a weapon's attachments as one word, and why those toggles are cheap. A rebuild that widens it must widen every save-game field that stores it. Saving and restoring the whole mask in one value is the point — the game persists it per entity.

### Driving the skeleton from outside

```text
FUNCTION add_bone_offset(offset : BoneOffset)
FUNCTION clear_bone_offset(bone_id)     # `none` clears every offset
```

**Contract** — a *bone offset* is a permanent correction applied to one bone after its pose is computed, every frame, until removed. `rotate_global` selects whether the rotation is composed in model-fixed axes or in the bone's own axes. Offsets accumulate: several may target the same bone and they apply in insertion order.

**Notes** — This is the script-facing hook for "this weapon sits two degrees off in this NPC's hand". It is a list scanned per bone per solve, so it is only cheap because it is almost always empty. A rebuild should consider attaching offsets to the bone record rather than scanning a side list, which is the same decision expressed better.

The per-bone `callback` on the bone instance is the *other* external driver and is different in kind: it runs inside the solve and can either post-process the computed pose (ragdoll blending, IK) or replace it entirely (`callback_overwrite`, used when physics owns the bone and animation would be wasted work). A driver that sets `callback_overwrite` is promising that it writes a valid transform; the solve does not compute one.

### `CalculateBones` — the solve

**Contract** — brings every visible bone's `transform` and `render_transform` up to date for the current instant, and periodically refreshes the model's world bounding volume. Idempotent within one instant. Blocks on a global skeleton lock. Allocates nothing. This is the single most-called expensive operation in the frame: every visible animated model runs it, and the renderer forces it on every model it is about to draw.

**Invariants** — after it returns, every visible bone's `transform` is consistent with its parent's, and `render_transform` matches.

```text
FUNCTION calculate_bones(force_exact : bool)
  IF now_ms == last_solve_ms
    RETURN                           # already solved this instant, for any reason

  LOCK global_skeleton_lock DURING
    on_calculate_bones()             # subclass hook: the animated skeleton advances its tracks here

    # The coarse rate. A skeleton nobody is looking at closely is allowed to be
    # up to UPDATE_INTERVAL stale; the renderer passes force_exact for anything
    # it is about to draw, and for anything a query is about to hit.
    IF NOT force_exact AND now_ms < last_solve_ms + UPDATE_INTERVAL
      RETURN

    IF visibility_dirty
      recompute_visible_bone_count()

    last_solve_ms = now_ms
    solve_subtree(root, identity)

    # The bounding box is rebuilt on a slower, deliberately desynchronized schedule.
    box_countdown = box_countdown + 1
    IF box_countdown >= BOX_REBUILD_PERIOD
      box_countdown = -random_int_below(BOX_REBUILD_PERIOD - 1)   # scatter the cost across models
      bounds = empty
      FOR EACH bone IN visible bones
        bounds = bounds UNION (bone.obb transformed by instance[bone].transform)
      model_sphere = bounding sphere of bounds
    IF update_callback EXISTS
      update_callback(self)          # the owner's chance to react to the new pose
```

```text
FUNCTION solve_subtree(bone, parent_transform)
  IF NOT bone_visible(bone)
    RETURN                           # the whole subtree is skipped; their transforms go stale
  instance = instance_of(bone)
  IF instance.callback_overwrite
    instance.callback(instance)      # the driver supplies the pose outright
  ELSE
    instance.transform = parent_transform . pose_of(bone)   # pose_of: bind, or blended animation
    apply_bone_offsets(bone, instance)
    IF instance.callback EXISTS
      instance.callback(instance)    # the driver corrects the pose in place
  instance.render_transform = instance.transform . bone.model_to_bone
  FOR EACH child IN bone.children
    solve_subtree(child, instance.transform)
```

**Notes** — Three timing decisions here are the reason the engine can afford hundreds of animated models.

`UPDATE_INTERVAL` is **100 ms — ten solves a second**. That is the floor rate for a skeleton nobody has asked about exactly. It is not a quality setting; it is the answer to "how stale may an off-screen creature's bones be before the AI's own bone queries go wrong", and shipped content is tuned around it.

The `now_ms == last_solve_ms` test at the top is a *different* early-out from the interval test, and both are needed: the first makes the operation free when it is called repeatedly within one instant, which it always is (the renderer, the physics step and several game queries each force it); the second makes it cheap when nobody needs precision.

The bounding-box countdown is seeded with a *negative random* value so that models loaded in the same frame do not all rebuild their boxes on the same later frame. Spreading a periodic cost by randomizing its phase rather than its period is a pattern worth keeping.

`force_exact` is the caller saying "I am about to read positions and stale is wrong". The renderer passes it before building a draw for the model; so does any code about to do a bone-accurate ray test. A rebuild that makes every solve exact is correct and slow; one that never forces it is fast and produces shots that miss visibly.

### Bone-accurate queries

```text
FUNCTION bone_pose(bone_id, channel_mask, ignore_drivers) -> Matrix
```

**Contract** — computes one bone's transform *without* disturbing the live skeleton, by walking that bone's ancestor chain only. `channel_mask` selects which animation channels contribute; `ignore_drivers` suppresses the per-bone callbacks so the caller sees the pose animation alone would produce. Used to ask "where would this bone be if physics were not holding it", which is how a ragdoll knows what to blend back towards.

```text
FUNCTION pick_bone(model_transform, ray_origin, ray_dir, max_dist, bone_id) -> optional<PickResult>
  # PickResult: surface normal, distance along the ray, the three triangle corners
```

**Contract** — intersects a ray against the *skinned triangles weighted to one bone*, in the bone's current pose. This is the exact hit test: the coarse pass finds a candidate bone from its box, this confirms the hit and yields the triangle, from which the game reads the surface material. Returns nothing on a miss. Requires the skeleton to be solved first.

```text
FUNCTION enumerate_bone_vertices(visitor, bone_id)
```

**Contract** — calls the visitor once per skinned vertex influenced by the given bone, in current pose. Used to fit a shape to a body part and to place wallmarks on skin.

**Notes** — Both of these force the renderer to skin on the processor even when the device would otherwise do all skinning, because the game needs the deformed positions and the device will not give them back. That is the load-bearing consequence: **a rebuild that skins only on the graphics device must still keep a processor-side skinning path for queries**, or lose hit detection precision on animated targets.

### Update notification

```text
FUNCTION set_update_callback(fn)          # called at the end of every full solve
FUNCTION set_update_callback_param(p)     # opaque payload carried back to the caller
FUNCTION update_callback() -> fn
FUNCTION update_callback_param() -> any
```

**Contract** — one callback per model, not per bone, fired after a solve completes. This is how an entity learns its pose changed and repositions attached objects, sound emitters and light sources. Setting it is not an ownership transfer; the callee must outlive the model or clear the callback first.

**Notes** — The getter/setter pair for an opaque payload is the C way of closing over state. A rebuild uses a closure and deletes four methods. What survives is the *timing*: the notification fires after the bones are final and after the bounding volume update, so a listener may read both.

### Downcasts

```text
FUNCTION as_visual() -> RenderVisual
FUNCTION as_animated() -> optional<AnimatedSkeleton>
```

**Contract** — the same object viewed as a renderable, and the same object viewed as an animation player if it is one. `as_animated` returning nothing is the normal answer for a rigid skeleton.

**Notes** — The engine navigates between these three views constantly and by hand, because the renderer's concrete model types are invisible outside the renderer. A rebuild with a real type system replaces the whole family with one interface and an optional capability; the decision that survives is that **"has a skeleton" and "plays animations" are separately queryable**, because game code branches on exactly that.

### Debug surface

```text
FUNCTION debug_render(model_transform)    # draws the bone hierarchy
FUNCTION debug_name() -> text             # the model's source path
```

Present only in a debug build, and every caller is already inside a debug guard. A rebuild may keep them unconditionally; the reason they are compiled out is that `debug_name` costs a string per model instance.

## `PKinematics` — the null-tolerant downcast helper

**Contract** — takes a possibly-absent renderable and returns its skeleton view, or nothing. A one-line convenience that exists because the engine asks this question in hundreds of places and half of them are on a value that may be absent.
