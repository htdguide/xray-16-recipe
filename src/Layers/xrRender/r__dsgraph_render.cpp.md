# src/Layers/xrRender/r__dsgraph_render.cpp

> Draining the buckets: opaque geometry ordered to minimise state changes and maximise early depth rejection, then each special bucket in its own required order.

**Needs** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`r__dsgraph_types.h`](r__dsgraph_types.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`FLOD.h`](FLOD.h.md) · [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrEngine/IRenderable.h`](../../xrEngine/IRenderable.h.md) · [`xrEngine/CustomHUD.h`](../../xrEngine/CustomHUD.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md)
**Used by** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md)
**Tier floor** — T1: it sorts and walks per-frame bucket arrays and issues draws into a command list; the sorts and the state-change accounting are the frame's second-largest cost after collection.

## Purpose

Where [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) decides *what*, this file decides *in what order*. Every function here is an ordering argument, and there are exactly two of them:

- **Opaque geometry** is ordered to make the device cheap: group by material pass so state changes are rare, and within that draw the biggest things first so the depth buffer rejects the most pixels earliest.
- **Everything else** is ordered because correctness demands it: transparency back to front, decals after their surfaces, the weapon layer under its own projection.

## The continuous detail parameter

```text
FUNCTION detail_level(coverage) -> real in [0, 1]
  RETURN sqrt(clamp((coverage - glod_end) / (glod_start - glod_end), 0, 1))
```

Computed per draw and handed to the visual, which uses it to choose how much of itself to emit — how many tree branches, how finely a progressive mesh is refined, how strong a normal map is. It is *continuous*, unlike the imposter switch, so there is no popping. The square root spreads the transition perceptually rather than by raw coverage. The same expression governs a shadowing light's contribution in [`light.cpp`](light.cpp.md), and the two must agree.

## `render_graph`

**Contract** — Draw one priority half of the opaque buckets, static then dynamic. Empties the buckets as it goes: a bucket drained is a bucket ready for next frame. Sets the world transform to identity at the start and relies on each dynamic draw to set its own.

```text
FUNCTION render_graph(priority_half)
  set world transform = identity

  FOR EACH of the two families (static, then dynamic)
    FOR EACH pass slot IN 0 .. max_passes
      passes = every (pass, bucket) pair in this slot

      # Order the PASSES. Two criteria, and the first overrides the second:
      # passes that are equivalent — same shaders, same textures, same state —
      # are treated as tied so the sort leaves them adjacent and the backend's
      # redundancy filter drops the change. Among non-equivalent passes, the
      # one holding the largest thing goes first.
      sort passes by: equal(a, b) ? tied : a.max_coverage >= b.max_coverage

      FOR EACH (pass, bucket) IN passes
        bind pass
        IF static family THEN apply the level's lighting material
        bucket.max_coverage = 0                # reset for next frame

        # Order the ITEMS within the pass, largest first, for depth rejection.
        sort bucket by coverage descending
        FOR EACH item IN bucket
          IF dynamic family
            set world transform = item.transform
            apply the owner's per-object constants
            apply the level's lighting material
          set the continuous detail parameter from item.coverage
          draw item.visual
        clear bucket
      clear the slot's map
```

**Invariants**

- The static family draws with the world transform left at identity from the top of the function — static geometry is authored in world space — and the dynamic family sets it per item. Interleaving the two families in one loop would require setting it either way and is why they are separate.
- The pass sort's equality test is a *content* comparison, not identity: two distinct pass objects that resolve to the same device state compare equal. Without it, materials that differ only in a name would each force a full state change.
- Buckets are cleared inside the loop, not afterwards, so a bucket's storage is released back for reuse as early as possible.
- The static family applies the level's lighting material once per pass; the dynamic family must apply it per item, because a dynamic object's lighting constants are its own.

## The first-person weapon layer

**Contract** — For as long as weapon-layer geometry is being drawn, the projection is replaced and the near plane is pulled in.

```text
DURING the HUD layer:
  save the current projection
  build a new projection from the HUD's own field-of-view multiplier, the
    screen aspect, a much nearer near plane, and the weather system's current
    far plane
  switch the device into its near-range depth configuration
  save the current cull mode
  ... draw ...
  restore the near-range configuration, the projection and the cull mode
```

**Invariants**

- The near plane is a *separate, nearer* constant than the world's. A weapon held at arm's length would otherwise clip through the near plane at every angle.
- The far plane comes from the weather system's current value rather than a constant, so the weapon layer's depth range matches the world's and the depth buffer's precision is not wasted.
- When the player is left-handed the weapon meshes are mirrored, which reverses their winding; the cull mode is flipped for the whole layer to compensate. The flip is applied per item, because the state is re-established by each pass bind.
- Field-of-view is currently a single global multiplier. The original carries an unimplemented plan for a per-weapon value; a rebuild wanting that must thread it from the visual to this bracket.

## `render_hud`, `render_sorted`, `render_emissive`, `render_wmarks`, `render_distort`

**Contract** — Drain one ordered bucket, in the direction its correctness requires, drawing each item with the element the collection pass chose for it, and clear the bucket.

| Bucket | Direction | Why |
|---|---|---|
| HUD | front to back | opaque; depth rejection |
| HUD sorted | back to front | transparent |
| sorted | back to front | transparent, against everything |
| emissive | front to back | additive; order is irrelevant, so take the cheap one |
| wallmarks | front to back | decals, already depth-biased onto their surfaces |
| distortion | back to front | refraction composes, so order matters |

Each item's draw is the same five steps: bind the item's element, set its transform, apply its owner's constants, apply the level's lighting material, apply the weapon-layer culling override if one is active, then draw at the detail level its coverage implies.

**Notes** — The emissive and wallmark buckets exist only on the deferred path. On the oldest renderer there is no accumulation buffer to add into and no decal pass, so both are absent entirely rather than empty.

## `render_hud_ui`

**Contract** — Draw the in-world user interface of the weapon the player is holding — a scope's reticle, a detector's screen. Runs under the same projection bracket as the rest of the weapon layer, but first **redirects the render targets**: the extra targets are unbound and the colour target is set to the accumulation buffer (or the colour buffer, depending on the deferred path's configuration) with the scene's depth buffer, or the multisampled depth buffer when multisampling is on.

The redirection is the point. This UI is drawn with ordinary forward shading into an already-lit buffer, so it must not land in the deferred path's geometry buffers — it has no normals and no material identity to contribute.

## `render_box`

**Contract** — Draw everything in one sector that intersects a given box, with a nominated shader element, ignoring visibility entirely. Used by the oldest renderer's dynamic-light pass, which re-draws the geometry inside a light's box.

```text
FUNCTION render_box(sector_id, box, element_slot)
  work = [ sector.root ]
  WHILE work is not empty
    take the next visual
    CASE its kind OF
      hierarchy, skeleton, level_of_detail:
        # For a skeleton, evaluate its bones first — this path can be reached
        # for a visual the collection pass never touched.
        push every child whose bounding box intersects the box
      anything else:
        element = visual.material.elements[element_slot]
        IF element exists AND is not a distortion element
          FOR EACH pass OF element
            bind it and draw the visual at the "no detail reduction" level
```

**Invariants** — The work list is walked by growing index rather than recursion, so a deep hierarchy cannot overflow a stack. The detail parameter is passed as a sentinel meaning "full detail": this pass is drawing into a light's volume and a reduced mesh would not match the depth already there.
