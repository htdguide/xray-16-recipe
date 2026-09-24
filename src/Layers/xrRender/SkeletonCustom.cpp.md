# src/Layers/xrRender/SkeletonCustom.cpp

> Loads a skeleton out of the shipped model file — bones, hierarchy, joint data, level-of-detail stand-in, the model's own configuration block — and owns the two things a posed skeleton does for the game: skinned decals and bone-accurate picking.

**Needs** — [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`SkeletonX.h`](SkeletonX.h.md) · [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`xrCore/Animation/Bone.hpp`](../../xrCore/Animation/Bone.hpp.md) · [`xrCDB/Intersect.hpp`](../../xrCDB/Intersect.hpp.md) · [`Include/xrRender/Kinematics.h`](../../Include/xrRender/Kinematics.h.md) · [`Shader.h`](Shader.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the bone hierarchy and the joint records are read as byte images out of a mapped model file, and the hierarchy is shared unowned between every instance of the model while the per-instance arrays are raw parallel allocations indexed by bone id.

## Purpose

This file implements the *rigid* skeleton — the model type that has bones but does not play animations. It is the base of the animated skeleton too, so everything here is also true of every animated creature in the game.

Three jobs, and the split between them is real:

1. **Load** the skeleton out of the model file and build the structures the rest of the engine indexes into: the bone array, the parent/child links, the two name lookup tables, the bind-pose-to-bone matrices, the visibility mask.
2. **Share** correctly. A model file is loaded once; every wearer gets a clone. The bone *descriptions* are shared and immutable; the bone *instances* are per-clone and mutable. Getting that division right is the whole reason the game can afford hundreds of skinned models.
3. **Serve queries against the posed skeleton**: ray-pick one bone's skinned triangles, and place and re-skin decals that stick to moving skin.

The pose computation itself — the solve — lives in [`SkeletonRigid.cpp`](SkeletonRigid.cpp.md), which is the same class continued in another file. That split is arbitrary; a rebuild may merge them.

## State

```text
RECORD Skeleton                          # extends the hierarchy visual: children are sub-meshes
  bones          : list<BoneData>        # SHARED with every clone of this model; immutable
  bone_instances : list<BoneInstance>    # PER-CLONE, parallel to bones, mutable
  root           : int                   # the one bone with no parent
  bone_map_by_name    : list<(text,int)> sorted by text          # SHARED
  bone_map_by_identity: list<(text,int)> sorted by string identity # SHARED
  user_data      : optional<Config>      # SHARED; the model's own ltx block
  visimask       : int (64-bit)          # PER-CLONE; bit b set => bone b participates
  lod            : optional<Visual>      # the far-distance stand-in model
  is_original_lod: bool                  # this instance is the loaded stand-in, not a clone of one
  children_invisible : list<Visual>      # sub-meshes parked because all their bones are hidden
  wallmarks      : list<SkeletonWallmark>
  wm_frame       : int                   # frame stamp: decals are re-submitted once per frame
  update_visibility : bool               # the sub-mesh partition needs recomputing
  solve_time_ms  : int                   # wall clock of the last pose solve
  box_countdown  : int                   # frames until the bounding volume is rebuilt
  bone_offsets   : list<BoneOffset>      # PER-CLONE; script-authored constant corrections
  update_callback: optional<fn>
```

```text
RECORD SkeletonWallmark                  # one decal stuck to skin
  parent         : Skeleton
  world_xform    : ref Matrix            # the wearer's transform, borrowed not owned
  shader         : Material
  contact_point  : Vector                # model space
  time_start     : real                  # global seconds, for the fade
  local_bounds   : Sphere                # model space
  bounds         : Sphere                # world space, refreshed each time it is drawn
  faces          : list<WMFace>

RECORD WMFace                            # one decal triangle, still in bind pose
  vert    : Vector[3]                    # bind-pose position of each corner
  uv      : Vector2[3]
  bone_id : int[3][4]                    # up to four influences per corner
  weight  : real[3][3]                   # three weights; the fourth is one minus their sum
```

**Invariants**

- **A model may have at most 64 bones.** The limit is the width of the visibility mask, it is asserted at load, and shipped content lives inside it. Bone *indices* are 16 bits, so the limit is purely the mask's; a rebuild that widens the mask must widen every place the game saves it.
- Exactly one bone has no parent, and it is the root. Two roots or none is a fatal content error, not a degenerate case.
- Bone names are **lowercased on load**. Every lookup in the engine, including the ones from Lua, therefore matches case-insensitively without anyone folding case at the call site.
- The bone array, both lookup tables and the configuration block are **owned by the base model and borrowed by every clone**. A clone that frees them corrupts its siblings; only the base's explicit release frees them.
- A hidden bone's transform is set to a **zero scale**, not left stale. Geometry weighted to it collapses to a point and disappears, which is what makes hiding work without touching the index buffer. The render transform is recomputed at the same moment, because the two must never disagree.
- A bone's render transform is always `transform` composed with the bone's model-to-bone matrix, and both are written together.
- The decal's `world_xform` is a *borrowed reference to the wearer's transform*, so a decal outlives nothing: the wearer must clear its decals before it dies.

## `Load`

**Contract** — reads a skeleton from an open model file: the level-of-detail stand-in, the configuration block, the bone table, the joint table, then the post-processing that links everything together. Blocks on file I/O and on loading the stand-in model. Fatal on a missing stand-in or a missing root bone.

```text
FUNCTION load(name, reader)
  load the hierarchy visual first            # the sub-meshes become our children

  IF chunk 23 (level-of-detail) EXISTS
    lod_name = one string
    lod = create_child_model(lod_name)
    mark that model as "the original stand-in"
    FAIL WITH fatal IF it could not be created

  IF chunk 17 (user data) EXISTS
    user_data = parse it as a configuration file, rooted at the game config path

  REQUIRE chunk 13 (bone names)
  count = one int ; FAIL WITH fatal IF count > 64
  FOR EACH of count bones
    name       = one zero-terminated string, lowercased
    parent     = one zero-terminated string, lowercased   # kept aside, resolved below
    obb        = one oriented box, as a byte image
    append a bone description with the next index; mark it visible
  sort both lookup tables
  link each bone to its parent by name; the parentless one is the root

  IF chunk 16 (joint data) EXISTS
    FOR EACH bone IN index order
      version          = one int              # per-bone, not per-file
      material name    = one string
      shape            = one byte image
      joint limits     = version-dependent decode
      bind rotation    = three reals, as Euler angles in a fixed axis order
      bind translation = three reals
      mass             = one real
      centre of mass   = three reals
    compute every bone's model-to-bone matrix by walking from the root

  FOR EACH sub-mesh child: let it bind itself to this skeleton by index
  FOR EACH bone, FOR EACH child: sort and deduplicate its face list
  clear the update callback, reset the decal frame stamp, validate breakables
```

**Invariants** — the joint chunk is optional; a skeleton without it has no masses, no materials and no bind transforms, and is only usable as a pose target. Every shipped creature has it.

**Notes**

- The **two lookup tables are the same data sorted two ways**, and both exist for speed, not for semantics: one is sorted by string content for lookups from a raw name, the other by the identity of the interned string for lookups from a name the caller already interned. The second is the one the game uses in anger — an interned-pointer comparison is a single integer compare, and bone lookup happens thousands of times per frame. A rebuild with a hash map keyed on the interned handle replaces both.
- The parent link is stored **by name** in the file, not by index, and resolved at load. That is what lets an artist reorder bones without renumbering anything, and it is why the resolve is a separate pass over the whole table.
- The **per-bone version field** inside the joint chunk is unusual and load-bearing: the joint-limit record grew over the engine's life and each bone records which layout it was written with. A rebuild must honour it per bone, not per file.
- The bind transform is stored as *Euler angles plus a translation*, not as a matrix. The axis order is fixed and frozen; get it wrong and every model is scrambled.
- The stand-in model is loaded through the *child* path, so it is not registered in the model cache under its own name and is not pooled. The "original stand-in" flag exists because a clone of this skeleton also clones the stand-in, and only the one that was actually loaded may be released — without the flag, the shared stand-in would be released once per clone.

## `LL_Validate` — the breakable-object consistency check

**Contract** — run once at load. If any bone is marked breakable, verify that the model's sub-meshes do not straddle the break boundaries; if they do, strip the breakable flag from every bone and (in a debug build) say so. No effect on a model with no breakable joints.

```text
FUNCTION validate_breakables()
  IF no bone has a real breakable joint
    RETURN

  # assign every bone a partition id: a new id starts at each breakable bone
  # and is inherited by its whole subtree
  partition_of = assign_partitions(root, id = 0)

  # a sub-mesh must be skinned entirely by bones from one partition
  FOR EACH sub-mesh group
    IF the group's bones do not all share a partition id
      strip breakable from every bone in the model
      RETURN
```

**Notes** — This is the engine refusing to half-support a piece of broken content. A breakable object is torn apart at a joint; if a single drawable sub-mesh spans both sides of the tear, there is no way to draw the two halves separately, and the result would be one half rendering geometry that belongs to the other. Rather than tear anyway, the model is demoted to non-breakable — it survives, it just never breaks. A rebuild should keep the check and the demotion: the shipped data contains models that fail it.

## `Copy` · `Spawn` · `Depart` · `Release`

**Contract** — the four lifecycle hooks the model cache drives. Together they define what is shared and what is private.

```text
FUNCTION copy(from)                 # making a clone from the base
  copy the hierarchy visual
  SHARE: bones, both lookup tables, user data, root index, visibility mask
  ALLOCATE: this clone's own bone instances
  tell every sub-mesh child that this skeleton is now its owner
  invalidate the pose
  duplicate the stand-in model, if there is one

FUNCTION spawn()                    # this clone is entering service
  reset every bone instance to its constructed state
  clear the update callback, invalidate the pose, clear decals
  invalidate the sub-mesh visibility partition
  set the root bone index to zero

FUNCTION depart()                   # this clone is leaving service, going to the pool
  clear decals
  make every bone visible again
  move every parked sub-mesh back into the drawn list

FUNCTION release()                  # the BASE model is being destroyed
  destroy every bone description, the lookup tables and the user data
```

**Invariants** — `Release` is the base's operation and frees shared data; `Depart` is a clone's and frees nothing. Calling `Release` on a clone destroys the data its siblings are still using.

**Notes**

- `Spawn` resetting the root bone index to **zero** rather than to the loaded root is a real inconsistency: the load pass finds the root by walking parent links and stores it, and then every spawn overwrites that with zero. It is harmless only because every shipped model happens to list its root bone first. A rebuild should keep the loaded value; a rebuild that matches the original bit-for-bit must also zero it.
- `Depart` un-hides every bone and un-parks every sub-mesh because a pooled clone will be handed to a different wearer, and per-wearer state — which limbs are severed, which attachments are on — must not survive the handover. This is the symmetric half of the model pool's reuse scheme.

## Bone visibility

**Contract** — hide or show a bone, optionally its whole subtree, or assign the whole mask at once. Hiding collapses the bone's transform to zero scale and marks the sub-mesh partition dirty; showing invalidates the pose so the next solve recomputes it.

```text
FUNCTION set_bone_visible(id, visible, recursive)
  visimask bit id = visible
  IF NOT visible
    bone_instances[id].transform = zero scale
  ELSE
    invalidate the pose
  bone_instances[id].render_transform = transform composed with model_to_bone
  IF recursive
    apply to every child, recursively
  mark the sub-mesh partition dirty
```

**Notes** — Note the asymmetry: hiding writes a pose immediately, showing only schedules one. Hiding must be immediate because the geometry is drawn from whatever is in the array; showing can wait because the solve will overwrite it anyway.

### `Visibility_Update` — parking whole sub-meshes

**Contract** — move each sub-mesh between the drawn list and the parked list according to whether any of its bones is still visible. Runs at most once per solve, and only when something changed the mask.

**Notes** — Hiding a bone already makes its geometry collapse to a point, so this is purely an optimization: a sub-mesh with no visible bones would still be skinned and still issue a draw call for triangles that all degenerate. Parking it skips both. The gain is real for the case it was built for — a corpse with severed limbs, where whole body sub-meshes go dark. Moving elements between two lists by swapping with the last is incidental; the decision that survives is **a sub-mesh whose every bone is hidden must not be skinned at all**.

## `AddWallmark` — sticking a decal to skin

**Contract** — given a ray in world space, find where it hits this skeleton's skinned geometry, and build a decal there: a set of triangles recorded *in bind pose*, with their skinning weights, so that the decal deforms with the body afterwards. Allocates. Does nothing if the ray misses. Called on every bullet impact on a creature.

```text
FUNCTION add_wallmark(wearer_xform, ray_start, ray_dir, material, size)
  transform the ray into model space

  # coarse then exact: the box test rejects almost every bone for almost every shot
  FOR EACH visible, pickable bone
    box = the bone's oriented box, posed by the bone's current transform   # cached
    IF ray misses box: CONTINUE
    FOR EACH sub-mesh: try an exact pick against this bone's triangles
      keep the nearest hit and its normal
  IF nothing was hit: RETURN

  contact = ray_start advanced by the nearest distance

  # which bones can possibly carry part of this decal
  affected = every visible pickable bone whose posed box meets the sphere (contact, size)

  # replace, do not stack: a second shot into the same spot with the same material
  # removes the first decal rather than adding a coplanar twin
  IF an existing decal has the same material and a contact within 0.02 of this one
    remove it

  decal = new decal (material, contact, now)
  decal.local_bounds = sphere(contact, size * 2)

  # the projection that generates texture coordinates
  blend_normal = normalize(surface_normal + reversed ray direction)
  projection   = look-at basis around blend_normal at contact, scaled by 1 / (0.9 * size)
  projection   = projection composed with a random rotation about its own axis, ±20 degrees

  FOR EACH sub-mesh, FOR EACH affected bone
    clip that bone's triangles against the projection and append them to the decal
  append the decal
```

**Notes**

- **The decal's triangles are stored in bind pose with their weights, not in world space.** That is the whole idea: a decal on a moving creature must move with the skin, and the only way to do that is to re-skin it every frame with the same influences its underlying vertices have. It is why a decal triangle carries four bone indices and three weights.
- The projection normal is the average of the surface normal and the *reversed shot direction*, not the surface normal alone. A grazing shot on a curved surface projects along something between the surface and the bullet path, which keeps the decal from smearing into a long thin streak.
- **The random twist of up to ±20 degrees** exists so that repeated hits in one area do not produce a visible grid of identically oriented decals. It is a texture-variety trick, and because it is random the same shot replayed does not produce the same image — an accepted non-determinism in a purely cosmetic system.
- The `0.9` in the projection scale gives the decal a 10% margin inside the requested radius, so the texture's edge does not land exactly on the clip boundary.
- The similarity check replaces rather than stacks, which caps decal count in the common case of sustained fire at one spot.
- The bone boxes are posed once into a scratch array and reused by both the pick loop and the sphere loop. That the array is sized to the whole bone count and mostly unused is incidental.

## `CalculateWallmarks`

**Contract** — once per frame, submit every decal that is still within its lifetime and inside the view to the renderer's decal collector. Expired decals are meant to be dropped here.

```text
FUNCTION calculate_wallmarks(is_hud)
  IF no decals OR already run this frame: RETURN
  stamp this frame
  FOR EACH decal
    age_fraction = (now - decal.time_start) / wallmark_lifetime
    IF age_fraction < 1
      IF NOT is_hud AND the decal's world sphere is possibly visible
        submit it to the renderer's decal collector
    ELSE
      mark that a removal pass is needed
```

**Notes** — Decals attached to the first-person view model are never submitted here; they are drawn by the view-model path with its own projection, and submitting them to the world collector would place them in the wrong space.

The default lifetime is **50 seconds** and is a console variable, adjustable from one second to ten minutes.

**A defect worth recording**: the removal pass removes decals that are *absent*, and nothing ever marks an expired decal as absent. Expired decals therefore stop being drawn but are never freed — they accumulate on a long-lived creature until it departs and its decal list is cleared wholesale. A rebuild should simply remove the expired entries.

## `RenderWallmark` — re-skinning a decal

**Contract** — writes one decal's triangles into a supplied vertex buffer cursor, skinned into the creature's current pose and transformed to world space, with a fade in the alpha channel. Advances the cursor. Requires the skeleton to be solved. Refuses if the skeleton's bone arrays have already been released.

```text
FUNCTION render_wallmark(decal, out cursor)
  fade = (now - decal.time_start) / wallmark_lifetime
  FOR EACH face IN decal.faces
    FOR EACH of the three corners
      # the influence count is encoded by repetition, not by a count field:
      # a slot equal to the next slot terminates the list
      IF bone[0] = bone[1]                       # one influence
        position = corner transformed by bone[0]'s render transform
      ELSE IF bone[1] = bone[2]                  # two influences
        position = lerp of the two transformed corners by weight[0]
      ELSE IF bone[2] = bone[3]                  # three influences
        position = weighted sum, third weight = 1 - weight[0] - weight[1]
      ELSE                                       # four influences
        position = weighted sum, fourth weight = 1 - sum of the first three
      write (decal.world_xform applied to position), the corner's texture
        coordinates, and colour grey with alpha = fade scaled to 0..255
  refresh the decal's world bounding sphere from its model-space one
```

**Invariants** — the weights always sum to one and the last one is never stored; reconstructing it by subtraction is what makes three weights enough for four influences.

**Notes**

- **The influence count is encoded by repeating the last bone index**, not by a separate count. It is a compact trick and it is frozen into how the decal records are built; the same convention appears in the skinned mesh formats. A rebuild may store a count instead as long as it does so consistently on both sides.
- The alpha ramps *up* with age — the decal starts transparent and becomes more opaque as it ages, reaching full opacity as it expires. That reads backwards for a fading decal and may be deliberate (blood spreading) or may be an inverted ramp nobody noticed; the source says nothing.
- The colour is a constant mid-grey. The decal's own colour comes entirely from its texture; the vertex colour exists to carry the alpha ramp and to give the material a neutral modulation term.

## `PickBone`

**Contract** — intersect a world-space ray with one named bone's skinned triangles and report the hit: normal, distance, and the triangle's three corners, all in world space. Delegates the actual test to each sub-mesh. Returns nothing on a miss.

**Notes** — The ray is transformed into model space once and the result transformed back, rather than transforming the geometry. The first sub-mesh that reports a hit wins; there is no nearest-of-all across sub-meshes, which is correct only because a bone's triangles almost never appear in two sub-meshes.

## `EnumBoneVertices`

**Contract** — visit every skinned vertex influenced by one bone, in the current pose, by forwarding to each sub-mesh. Used to fit a shape to a body part.

## `LL_GetBindTransform`

**Contract** — produce every bone's bind pose in model space, by composing bind transforms down the hierarchy from the root. Allocates the output. Used by the skinning setup and by anything that needs the rest pose.

## `LL_GetBoneGroups`

**Contract** — report, for each sub-mesh, the list of bones that skin any of its faces. Derived from the per-bone face lists built at load.

**Notes** — This is how the renderer learns each draw call's bone palette and how the game learns which sub-mesh a body part lives in. That it is recomputed on demand rather than stored is a small waste; it is called rarely.

## `LL_BoneID` · `LL_BoneName_dbg`

**Contract** — name to index through whichever of the two sorted tables matches the caller's string form; missing names return the sentinel rather than failing. The reverse direction is a linear scan and exists only for diagnostics.

## `DebugRender`

**Contract** — debug builds only: solve the skeleton, then draw a line for every parent-child link, a small marker box at every joint, and each bone's oriented box. The bone hierarchy is queried from the root as a flat list of index pairs.
