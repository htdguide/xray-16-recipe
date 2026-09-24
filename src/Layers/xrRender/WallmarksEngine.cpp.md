# src/Layers/xrRender/WallmarksEngine.cpp

> Decals: how a bullet hole is cut out of the world's collision geometry, how it is batched by material, and how it fades and dies.

**Needs** — [`WallmarksEngine.h`](WallmarksEngine.h.md) · [`SkeletonCustom.h`](SkeletonCustom.h.md) · [`Shader.h`](Shader.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrEngine/Render.h`](../../xrEngine/Render.h.md) · [`xrEngine/xr_object.h`](../../xrEngine/xr_object.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes interleaved vertices straight into a mapped device buffer and owns the budget that keeps the write in bounds.

## Purpose

A **wallmark** is a decal — a bullet hole, a blood splash, a scorch mark — projected onto whatever surface a shot landed on. It is not a texture on the surface and it is not a quad floating in front of it: it is a *new piece of geometry*, cut from the surface's own triangles, so that it wraps around corners and follows the ground exactly. This file cuts it, stores it, and draws it.

Two kinds coexist and are drawn by the same batching loop:

- **static** wallmarks, cut once out of the level's static collision triangles and then owned here until they fade out;
- **skeleton** wallmarks, attached to a bone of an animated model, re-skinned every frame by the model itself and thrown away at the end of each frame.

There is a third kind the engine does not own here at all — wallmarks baked into the level's own geometry — which is drawn by the draw stream at the end of this pass.

## State

```text
RECORD StaticWallmark
  bounds : sphere                  # for culling and for the similarity test
  verts  : list<Vertex>            # position + colour + uv, triangle list, already clipped
  ttl    : real                    # seconds of life remaining

RECORD Slot                        # one material's worth of wallmarks
  shader          : Shader         # the material every item in this slot is drawn with
  static_items    : list<StaticWallmark>
  skeleton_items  : list<SkeletonWallmark>

RECORD WallmarksEngine
  slots       : list<Slot>         # linear; one per distinct material in use
  static_pool : list<StaticWallmark>   # freed records, kept for reuse
  geometry    : GeometryHandle     # one shared vertex format and buffer for all of them
  lock        : Mutex
```

**Invariants**

- Slots are looked up **by material identity**, linearly, and there is exactly one slot per material. That is the whole batching strategy: one buffer fill and one draw call per material per frame, however many marks it holds.
- A static wallmark's record is never freed, only returned to the pool and reused. Cutting a wallmark happens in response to a bullet and must not allocate on that path.
- `skeleton_items` is cleared at the end of every frame's render. A skeleton wallmark is re-submitted by its owning model each frame it is visible; one that stops being submitted simply stops being drawn. Static marks are the opposite — they live in this module until their time expires.
- Wallmarks are added from the physics side while the renderer is drawing them, so every mutation of the slot list and every traversal of it is taken under one lock.

## `add_static_wallmark(triangle, vertices, contact_point, material, size)`

**Contract** — cuts a decal out of the static world at a contact point and files it under the given material. Does nothing at all when the contact is more than 100 metres from the camera. Takes the lock. Allocates only on the pool's first miss.

**Notes** — The distance cutoff is stated in the source as a cheat and it is one, but it is the right kind: the cut is an expensive collision query plus a recursive clip, and a decal 100 metres away is sub-pixel. The number is a straight trade of fidelity for the cost of a firefight happening off-screen.

## `add_static_wallmark` — the cut

**Contract** — the algorithm that turns a point and a size into wrapped geometry. Fatal on nothing; gives up quietly whenever the result would be degenerate.

```text
FUNCTION cut_wallmark(hit_triangle, world_vertices, contact_point, material, size)

  # 1. Gather the neighbourhood. The decal may span several triangles, so
  #    collect every static triangle within a box around the contact and
  #    build an adjacency table over just those.
  box := cube centred on contact_point, half-extent size * 2.5
  hits := static_collision.query_box(box, exact = true)
  IF hits is empty THEN RETURN

  local := new TriangleCollector
  local.add(hit_triangle)                 # index 0 is ALWAYS the triangle that was hit
  FOR EACH t IN hits WHERE t is not hit_triangle
      local.add(t)
  adjacency := local.compute_adjacency()

  # 2. Build a projector looking along the hit triangle's normal, scaled so
  #    that one unit of the projection is one wallmark.
  normal := face_normal(hit_triangle)
  view   := camera_looking_at(contact_point, along = -normal) scaled by 1/size
  view   := view rotated about its own axis by a random angle in [-20, +20] degrees
  clipper := frustum_from(view, side planes only)   # no near or far plane

  # 3. Flood-fill outward from the hit triangle, clipping each face to the
  #    projector and emitting its fan.
  mark := pool.take()
  mark.ttl := wallmark_lifetime
  recurse(0, view, mark)

  # 4. A decal that came out as fewer than one triangle is not a decal.
  IF mark.verts.size < 3 THEN pool.give_back(mark) ; RETURN
  mark.bounds := bounding_sphere(mark.verts)

  # 5. Coalesce: a new mark whose centre is within 2 cm of an existing mark
  #    of the SAME material replaces it rather than stacking on it.
  slot := find_slot(material) OR append_slot(material)
  FOR EACH existing IN slot.static_items
      IF existing.bounds.centre is within 0.02 of mark.bounds.centre THEN
          pool.give_back(existing) ; replace it with mark ; RETURN
  slot.static_items.append(mark)
```

**Notes** — The random rotation in step 2 is the single decision that makes repeated fire look like damage instead of like wallpaper: without it every bullet hole lands axis-aligned and a wall of them reads as a grid. ±20° is enough to break the pattern and small enough that a directional mark (a blood streak) still points roughly where it was authored to.

Step 5 is what keeps a burst of automatic fire from stacking a hundred coincident decals into one z-fighting smear, and it is also the population control: without it the oldest marks would be pushed out by sheer count long before their lifetime expired.

The neighbourhood box is 2.5 times the decal's size rather than 1, because the decal is rotated and because a triangle may reach into the projection from well outside it.

## `recurse(triangle, projector, mark)` — the flood fill

**Contract** — clips one triangle to the projector, emits its fan, and walks to its neighbours. Marks each triangle visited so the walk terminates. Appends to the mark's vertex list; allocates as that list grows.

```text
FUNCTION recurse(t, projector, mark)
  IF t already visited THEN RETURN
  mark t visited

  poly := clip(triangle t, projector)        # side planes only
  IF poly is empty THEN RETURN

  # Texture coordinates come from the projection itself: a vertex's
  # position through the projector lands in [-1,1], which maps to [0,1]
  # with the vertical axis flipped. There is no authored UV anywhere.
  FOR EACH vertex v IN poly
      p := projector.transform(v)
      v.uv := ( (1 + p.x) * 0.5 , (1 - p.y) * 0.5 )
  emit poly as a triangle fan into mark.verts

  FOR EACH neighbour n OF t
      IF n does not exist THEN CONTINUE
      # Stop at a fold. A neighbour whose normal turns more than 88 degrees
      # away is a different surface, and continuing onto it wraps the decal
      # around a corner it should not see.
      IF dot(face_normal(n), hit_normal) < cos(88 degrees) THEN CONTINUE
      recurse(n, projector, mark)
```

**Invariants** — the walk starts at the hit triangle and never leaves the neighbourhood collected in step 1, so its depth is bounded by that triangle count. The 88° threshold is expressed as a cosine compared against, and it lets the decal cross a nearly-perpendicular join (a floor meeting a shallow ramp) while refusing a true corner.

**Notes** — The `visited` mark is written into a scratch field of the collected triangle rather than into a side set, which is fine because the collection is rebuilt for every cut; a rebuild with an immutable triangle store needs a visited set instead.

Clipping uses only the projector's four side planes. There is no near or far plane, deliberately: a decal must be allowed to reach an arbitrary distance along the surface normal (down a slope, into a recess) and clipping in depth would cut it short.

## `add_skeleton_wallmark(transform, model, material, start, direction, size)`

**Contract** — asks an animated model to cut a decal into *its own* skinned geometry. Refuses beyond 50 metres — half the static cutoff, because a skinned decal costs a re-skin every frame it lives, not once. Takes the lock, because the model's own wallmark list is walked by the render pass.

## `add_skeleton_wallmark(mark)`

**Contract** — files an already-cut skeleton wallmark under its material, creating the slot if needed. Called by the model once per frame for each of its live decals. No coalescing and no lifetime: the model owns both.

## `render()`

**Contract** — draws every slot. Fills a shared dynamic vertex buffer, flushing whenever the next item would not fit, and issues one draw per fill. Ages static marks and destroys the expired. Takes the lock for the whole traversal. Leaves the view and projection transforms as it found them.

```text
FUNCTION render()
  # The depth bias is applied to the PROJECTION and to the VIEW POSITION,
  # not to the rasterizer: the projection's depth row is nudged and the
  # camera is walked forward along its own direction. Doing it this way
  # biases in world units rather than in depth-buffer units, so a decal
  # sits the same distance off the wall at every range.
  projection := device.projection with its depth offset reduced by wallmark_depth_shift
  view       := camera moved forward by wallmark_view_shift
  set world = identity, view, projection

  cull_threshold := screen_area_discard_threshold / 4   # decals survive
                                                        # four times smaller than models

  LOCK slot list DURING
    FOR EACH slot IN slots
      begin_stream()

      # --- static: cull, age, emit ---
      FOR EACH mark IN slot.static_items
        IF mark.bounds intersects the view frustum THEN
            screen_area := mark.bounds.radius^2 / distance_to_camera_squared
            IF screen_area >= cull_threshold THEN
                IF stream would overflow with mark.verts THEN flush ; begin_stream()
                emit mark with colour = grey, alpha = fade(mark.ttl)
            mark.ttl := mark.ttl - 0.1 * frame_time     # visible: ages slowly
        ELSE
            mark.ttl := mark.ttl - frame_time           # unseen: ages at full rate
        IF mark.ttl <= 0 THEN
            pool.give_back(mark)
            swap the last item into this slot and shrink the list
            BREAK                                        # see Notes
      flush(cull_backfaces = true)

      # --- skeleton: same batching, no ageing, no culling by frustum ---
      begin_stream()
      FOR EACH mark IN slot.skeleton_items
        screen_area := mark.bounds.radius^2 / distance_to_camera_squared
        IF screen_area < cull_threshold THEN CONTINUE
        IF stream would overflow with mark.vertex_count THEN flush(no cull) ; begin_stream()
        ask the owning model to skin the mark into the stream
      flush(cull_backfaces = false)
      slot.skeleton_items.clear()

  draw the level's own baked wallmarks
  restore the device's view and projection
```

**Invariants**

- The stream is bounded at **16384 triangles** per fill. The cap exists because the whole fill is one mapped region of a dynamic buffer sized once; a single explosion against a dense mesh can produce far more decal geometry than any normal frame, and without the cap the write runs off the end of the mapping.
- Skeleton wallmarks are drawn with backface culling **off** and static ones with it on. A skinned decal follows a deforming surface and can legitimately end up facing away from the camera in part; a static one is cut from front-facing geometry by construction.
- The slot's material is set once per flush, which is the point of grouping by material at all.

**Notes** — The ageing asymmetry is the interesting decision: a wallmark the player can see decays at one tenth the rate of one they cannot. The effect is that decals persist while you are looking at the firefight and evaporate behind your back, which keeps the visible count bounded without the player ever seeing one vanish. A rebuild that ages everything uniformly will either run out of budget or make marks disappear on screen.

The expiry path removes one mark and then **stops scanning that slot for the frame** — at most one static wallmark per material dies per frame. That is not obviously intentional: the loop is iterating the same list it is mutating, and breaking is the cheap way to stay safe. The consequence is a bounded, slow drain rather than a cliff, which is benign; a rebuild is free to remove all expired marks in one pass, and should, but should keep the ageing rates above or the visible population will change.

The skinning call is wrapped so that a failure to skin one decal loses that decal and nothing else — the stream cursor is rewound to where it was. That is a guard against malformed model data reached through the physics path, not against a condition the renderer can cause.
