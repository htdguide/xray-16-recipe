# src/Layers/xrRender/r__dsgraph_build.cpp

> Collection: walk the sector/portal topology and the spatial database, descend every visible visual to its drawable leaves, and file each leaf in the bucket its material demands.

**Needs** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`r__dsgraph_types.h`](r__dsgraph_types.h.md) · [`r__sector.h`](r__sector.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`FHierrarhyVisual.h`](FHierrarhyVisual.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`FLOD.h`](FLOD.h.md) · [`FTreeVisual.h`](FTreeVisual.h.md) · [`ParticleGroup.h`](ParticleGroup.h.md) · [`LightTrack.h`](LightTrack.h.md) · [`HOM.h`](HOM.h.md) · [`light.h`](light.h.md) · [`xrEngine/IRenderable.h`](../../xrEngine/IRenderable.h.md) · [`xrEngine/CustomHUD.h`](../../xrEngine/CustomHUD.h.md) · [`xrCDB/ISpatial.h`](../../xrCDB/ISpatial.h.md) · [`xrRender_console.h`](xrRender_console.h.md)
**Used by** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md)
**Tier floor** — T1: the hot loop of the frame; it walks pointer hierarchies, stamps per-context markers in place and appends to preallocated buckets without allocating.

## Purpose

This is the renderer's core: the pass that decides what will be drawn this frame. It runs once per render context. Everything downstream — the depth prepass, the light accumulation, transparency, the shadow maps — drains the buckets this file fills.

It is worth naming the three filters it applies, in order, because a rebuild must apply them in the same order for the same reason:

1. **Topology** — the sector/portal walk yields the reachable sectors and a clipped frustum per route into each. Cheap, and it cuts the world down to a room or two.
2. **Occlusion** — each candidate's bounding volume is tested against the software occlusion map. Cheap, conservative, and cuts what the portals let through.
3. **Coverage** — the screen-space-area estimate rejects anything too small to see and chooses the detail level of what remains. Almost free, and it is what makes a view down a long street affordable.

## The coverage estimate

```text
FUNCTION coverage(centre, radius) -> (real, real)
  distance_squared = camera_position.distance_squared_to(centre) + epsilon
  RETURN (radius / distance_squared, distance_squared)
```

This single expression is used by every decision in the renderer that depends on apparent size. It is not an area and not an angle: it is radius over squared distance, which is proportional to solid angle for small objects and is cheap. Every threshold in the engine is calibrated against *this* expression, so a rebuild that substitutes a truer measure must recalibrate every one of them.

The thresholds it feeds, all runtime-configurable:

```text
discard_coverage   # below this, a visual is not drawn at all
lod_enter_coverage # below this, a level-of-detail imposter is drawn
lod_exit_coverage  # above this, the real geometry is drawn
                   # between the two, BOTH are drawn and cross-faded
glod_start, glod_end  # the range over which a visual's continuous detail
                      # parameter goes from 1 to 0
dont_sort_coverage    # above this, ordering within a pass stops mattering
```

## `insert_static` and `insert_dynamic` — the funnel

**Contract** — File one drawable leaf in the right bucket, or reject it. Every path through this file ends in one of these two. They differ only in that the dynamic form carries an owner and a transform; the bucket decision is identical and is the substance.

```text
FUNCTION insert(visual, owner, transform, centre)
  # 1. Once per context per frame.
  IF visual.marker[context_id] = marker THEN RETURN
  visual.marker[context_id] = marker

  # 2. Too small to see.
  (cov, dist_sq) = coverage(centre, visual.bounding_radius)
  IF cov <= discard_coverage THEN RETURN

  # 3. Distortion is a SEPARATE registration, not an alternative one: a
  #    surface that refracts is drawn once into the distortion buffer and
  #    again normally. The element it uses is the material's "special" slot.
  special = visual.material.elements[special_slot]
  IF distortion supported AND special declares distortion
     AND priority_mask admits special.priority
    distortion.insert(key = dist_sq, item with element = special)

  # 4. Choose the element for this phase and this distance. The renderer
  #    picks a level-of-detail element here: a near object gets the full
  #    material, a far one a cheaper variant. A phase may have no element at
  #    all — a shadow pass has nothing to draw for a material that casts none.
  element = select_element(visual, dist_sq, phase)
  IF element is none THEN RETURN
  IF NOT priority_mask[element.priority / 2] THEN RETURN

  # 5. The first-person weapon layer is its own world: a different projection
  #    and a different near plane, applied once around the whole bucket rather
  #    than per draw. It therefore cannot share a bucket with anything else.
  IF owner is the HUD
    IF element demands strict back-to-front
      hud_sorted.insert(dist_sq, item) ; RETURN
    hud.insert(dist_sq, item)
    IF element is emissive THEN hud_emissive.insert(dist_sq, item with special element)
    RETURN

  IF owner says it is currently invisible THEN RETURN

  # 6. Strict transparency: must be drawn back to front against everything
  #    else transparent, so it cannot be grouped by pass.
  IF element demands strict back-to-front
    sorted.insert(dist_sq, item) ; RETURN

  # 7. Emissive surfaces are ALSO drawn normally; the emissive bucket is an
  #    additional accumulation pass that makes them self-lit. Decals, in
  #    contrast, are drawn ONLY from their own bucket, because they must land
  #    after the surfaces they sit on.
  IF element is emissive THEN emissive.insert(dist_sq, item with special element)
  IF element is a wallmark AND decals wanted
    wallmarks.insert(dist_sq, item) ; RETURN

  # 8. The ordinary case: one entry per pass of the element, in the bucket
  #    keyed by that pass. Each bucket tracks the largest coverage it holds,
  #    which is what later orders the passes against each other.
  FOR EACH pass, slot IN element.passes
    bucket = opaque[element.priority / 2][slot][pass]
    bucket.append(item)
    bucket.max_coverage = max(bucket.max_coverage, cov)

  IF box_recorder is set THEN record the visual's world-space box
```

**Invariants**

- The marker stamp happens *before* any rejection, so a visual rejected on one route is not reconsidered on another. That is intentional: the cheapest rejection should be the one that sticks.
- Several buckets may receive the same visual in one call — distortion, emissive and an opaque bucket — and each holds a different element. A rebuild must not "optimise" this into one entry.
- Early returns are load-bearing. Strictly sorted, HUD-sorted and wallmark registrations replace the opaque registration; emissive and distortion supplement it.

## `add_leafs_static` and `add_leafs_dynamic` — descent without testing

**Contract** — Given a visual already known to be fully visible, walk it to its drawable leaves and funnel each. No frustum test is performed anywhere below this point; that is what "leafs" means.

The descent is by visual kind, and each kind's rule is a decision:

```text
FUNCTION add_leafs(visual)
  IF static AND occlusion map enabled AND visual's volume is occluded THEN RETURN

  CASE visual.kind OF
    particle_group:
      # A group holds effects, each with related and free child systems.
      # Descend into all three lists unconditionally; particles are small and
      # testing each is more expensive than drawing it.
      # (Only reachable on the dynamic path — a particle group has no world
      #  transform of its own, so reaching one via the static path is a bug
      #  and the original treats it as fatal.)

    hierarchy:
      descend into every child

    skeleton (animated or rigid):
      # The expensive case, and the only place bones are evaluated.
      IF the visual has an imposter AND its coverage is below the enter
         threshold      # note: coverage is measured with HALF the bounding
                        # radius, because an animated figure never fills its
                        # own bounding sphere
        descend into the imposter instead
      ELSE
        evaluate the skeleton's bones NOW
        IF this is the normal phase THEN also update its decals
        descend into every child

    level_of_detail (an imposter node):
      cov = coverage(...) * the node's own authored factor
      IF cov < enter_threshold
        IF cov < discard_threshold THEN RETURN
        queue the imposter, keyed by distance
      IF cov > exit_threshold OR this is a shadow pass
        descend into the real children as well
      # Between the two thresholds BOTH are queued and cross-faded by alpha;
      # a shadow pass always uses the real geometry, because an imposter's
      # silhouette is a camera-facing billboard and would cast a flat shadow.

    anything else:
      funnel it
```

**Invariants**

- Bone evaluation happens *during collection*, not during drawing. By the time a skinned visual is in a bucket its bone matrices are final. This is why a visual may be collected only once per context — evaluating twice would be wasted, and evaluating after the buckets were filled would be too late.
- Decals are updated only in the normal phase. A shadow pass must not, because decal geometry is generated against the camera and a shadow camera would regenerate it wrongly. The original marks the condition it uses for the HUD flag here as questionable; treat the flag as "which layer's decals" rather than anything deeper.
- The imposter's coverage uses the *node's own* factor, an authored per-imposter multiplier that lets a level designer make a particular distant object switch earlier or later.

## `add_static` — descent with testing

**Contract** — The same descent, but each node is first tested against the frustum and the occlusion map, and a node found *fully* inside stops testing its children and falls through to the leaf descent. A node found partially inside keeps testing.

```text
FUNCTION add_static(visual, frustum, active_planes)
  visibility = frustum.test_sphere_then_box(visual.volume, active_planes)
  IF fully outside THEN RETURN
  IF occlusion map enabled AND occluded THEN RETURN

  CASE visual.kind OF
    hierarchy, skeleton:
      IF partially inside THEN test each child   # keep clipping
      ELSE                     descend without testing each child
    level_of_detail: same thresholds as above
    anything else:  funnel it
```

**Notes** — The frustum test takes a set of *active planes* and returns which planes the node still straddles, so a child is tested only against the planes its parent did not already resolve. This is the standard optimisation and it is what makes a deep static hierarchy cheap. The sphere is tested before the box because a sphere test is three dot products.

## `load` and `unload`

**Contract** — Build this context's sectors and portals from the level's parsed visibility data, in two passes: create every portal and every sector first, giving each sector its identifier, its portal list and its root visual; then wire each portal to the two sectors it separates. Two passes because the references are mutual. Unloading destroys both lists.

On a dedicated server a sector gets no root visual — there is no geometry to draw — but the topology is still built, because the server uses sectors to answer "where is this entity".

## `build_subspace` — the collection pass

**Contract** — Fill every bucket of this context from scratch. Reads the level's topology, the spatial database and the occlusion map; writes this context's buckets and, on the main pass, updates each spatial object's recorded sector and each renderable's lighting track. Called once per context per frame.

```text
FUNCTION build_subspace()
  marker += 1          # critical: everything downstream keys off this

  # --- Near-portal handling -------------------------------------------
  # A camera standing in a doorway is on neither side of that portal. Any
  # portal within a small box of the view point is flagged DUAL, which makes
  # the traversal enter it from both sides instead of choosing one.
  IF precise_portals
    FOR EACH portal whose triangle meets a small box around view_pos
      portal.dual = true

  # --- Degenerate case ------------------------------------------------
  # The camera is outside the sector structure entirely (in the void, or
  # between levels). Nothing static is visible; only the weapon layer draws.
  IF is_main_pass AND sector_id is invalid
    render the HUD's late layer ; RETURN

  # --- 1. Topology ----------------------------------------------------
  traverser.traverse(sectors[sector_id], view_frustum, view_pos, xform,
                     portal_traverse_flags)

  # --- 2. Static geometry ---------------------------------------------
  # Each reached sector may have been reached by several routes, each with
  # its own clipped frustum. The sector's root is walked once per route.
  IF static drawing enabled
    FOR EACH sector IN traverser.reached
      FOR EACH clipped frustum OF that sector
        add_static(sector.root, frustum, frustum.active_planes)

  # --- 3. Dynamic objects and lights ----------------------------------
  IF dynamic drawing enabled AND any spatial type wanted
    candidates = spatial_database.query_frustum(view_frustum, spatial_types,
                                                spatial_traverse_flags)
    IF ordered was requested
      sort candidates front-to-back by squared distance to view_pos

    # Light-tracking: exactly ONE object per frame has its lighting
    # environment re-sampled, chosen round-robin. Sampling is expensive and
    # lighting changes slowly, so the cost is amortised across objects. The
    # object the player is viewing from is always updated as well.
    IF is_main_pass AND phase is normal
      tracked_index = (frame counter) modulo candidates.count
      update the view entity's lighting track

    FOR EACH candidate
      IF is_main_pass
        candidate.sector = detect_sector(candidate.sector_point)
      IF candidate.sector is invalid THEN CONTINUE   # outside the topology

      IF candidate is a light source AND lights wanted
        IF its level-of-detail factor is above zero
           AND its volume passes the occlusion map
          add it to the frame's light list
        CONTINUE

      IF candidate's sector was not reached by the traversal THEN CONTINUE

      FOR EACH clipped frustum OF that sector
        IF the candidate's bounding sphere fails this frustum THEN CONTINUE
        IF is_main_pass
          # Occlusion-test the object's TRANSFORMED box, then copy the test's
          # bookkeeping back into the object's own record — the test is
          # performed on a copy because the transform must not be baked into
          # the visual, but its cached results must survive.
          IF occluded THEN BREAK out of the frustum loop
          IF this is the round-robin object THEN re-sample its lighting
          ask the object to render itself into this context
          BREAK out of the frustum loop     # collected once, from one route
        ELSE
          ask the object to render itself into this context

  # --- 4. The player's own shadow -------------------------------------
  # The view entity is normally not drawn at all in first person, so it casts
  # no shadow. When actor shadows are enabled, the shadow-map phase explicitly
  # collects the weapon layer's early part for exactly this purpose.
  IF phase is shadow_map AND actor shadows enabled
    locate the view entity's sector, test it against that sector's frustums,
    and collect the HUD's early layer

  IF is_main_pass THEN collect the HUD's late layer
```

**Invariants**

- "Ask the object to render itself into this context" is a callback: the object decides which visual, which transform and whether to split into several, then calls back into `add_leafs_dynamic`. The graph never reaches into an object's internals.
- An object is collected from exactly one route on the main pass (the loop breaks) but from *every* route on other passes. That is deliberate: the main pass draws each object once, while a shadow pass may legitimately need the same object relative to several clipped frusta.
- The occlusion test on the main pass operates on a copy of the visibility record with the transform applied, and then copies the cached test results back. Without the copy-back the object would be re-tested from scratch every route; without the copy the transform would corrupt the visual's authored bounds.
- Light sources short-circuit before the frustum loop: a light is not drawn, it is *collected*, and its own visibility testing is [`light_vis.cpp`](light_vis.cpp.md)'s job.

**Notes** — A parallel version of the static-geometry walk is present but disabled, with a note explaining why: the buckets are shared across the walk and would need to be partitioned per worker before the walk could be split. That is the real obstacle to multithreading the collection pass, and it is worth knowing before attempting it in a rebuild.
