# src/Layers/xrRender/r__dsgraph_structure.h

> One render context: the visibility options it was given, the sector/portal topology it walks, the buckets it fills, and the command list it drains them into.

**Needs** — [`r__dsgraph_types.h`](r__dsgraph_types.h.md) · [`r__sector.h`](r__sector.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrCDB/ISpatial.h`](../../xrCDB/ISpatial.h.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) · [`r__dsgraph_render_lods.cpp`](r__dsgraph_render_lods.cpp.md) · [`r__sector_detect.cpp`](r__sector_detect.cpp.md) · [`Include/xrRender/RenderVisual.h`](../../Include/xrRender/RenderVisual.h.md)
**Used by** — [`D3DXRenderBase.h`](D3DXRenderBase.h.md) · [`light_smapvis.cpp`](light_smapvis.cpp.md) · [`light_smapvis.h`](light_smapvis.h.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) · [`r__dsgraph_render_lods.cpp`](r__dsgraph_render_lods.cpp.md) · [`r__dsgraph_types.h`](r__dsgraph_types.h.md) · [`r__sector_detect.cpp`](r__sector_detect.cpp.md)
**Tier floor** — T1: it is a large mutable per-context aggregate whose buckets are reused across frames to avoid reallocating, and whose per-visual marker is an index into a fixed-size per-context array.

## Purpose

Declares the central working object of the renderer. A *render context* is one complete "collect what is visible and draw it" job: the main camera's pass is one, each shadow map is another, a reflection is another. Several exist and several may be built concurrently.

Everything about a context is here: what it was asked to collect, what it found, and where it will send it. The algorithms are split across four files by phase — [building](r__dsgraph_build.cpp.md), [rendering](r__dsgraph_render.cpp.md), [imposters](r__dsgraph_render_lods.cpp.md) and [sector detection](r__sector_detect.cpp.md) — and that split is arbitrary, a consequence of file size rather than of design. A rebuild may merge them.

## State

```text
RECORD RenderContext
  # --- identity -----------------------------------------------------------
  context_id : int          # index into every visual's per-context marker array;
                            # the immediate context sits just past the pooled ones
  marker     : int          # advanced once per build; stamped into each visual it
                            # accepts, so a visual reachable by several routes is
                            # collected once

  # --- what to collect ----------------------------------------------------
  options :
    phase                  : ENUM { normal, shadow_map, ... }
    portal_traverse_flags  : set of { use_occlusion_map, use_coverage,
                                      use_scissor, fade_portals }
    spatial_traverse_flags : set of { ordered, ... }
    spatial_types          : set of { renderable, light_source }
    query_box_side         : real    # how near a portal must be to force dual rendering
    view_pos               : vector3
    xform                  : matrix4 # the combined view-projection
    view_frustum           : frustum
    sector_id              : sector_id
    priority_mask          : list<bool> of 2   # which priority halves to admit
    admit_wallmarks        : bool
    use_occlusion_map      : bool
    precise_portals        : bool
    is_main_pass           : bool
    multithreaded          : bool

  # --- the topology (owned per context, loaded from the level) -------------
  sectors   : list<Sector>
  portals   : list<Portal>
  traverser : PortalTraverser
  ray_cache : CollisionQueryContext

  # --- what was found -----------------------------------------------------
  opaque_static  : StaticPasses  indexed by priority half
  opaque_dynamic : DynamicPasses indexed by priority half
  sorted         : SortedBucket    # strict back-to-front transparency
  hud            : SortedBucket    # first-person weapon layer, opaque
  hud_sorted     : SortedBucket    # first-person weapon layer, transparent
  lods           : LodBucket       # distant imposters
  distortion     : SortedBucket    # heat haze and similar
  wallmarks      : SortedBucket    # decals (deferred path only)
  emissive       : SortedBucket    # self-lit surfaces (deferred path only)
  hud_emissive   : SortedBucket

  # --- reusable scratch ---------------------------------------------------
  pass_scratch, lod_scratch, spatial_scratch, visual_scratch : lists

  # --- instrumentation and hooks ------------------------------------------
  feedback           : optional<CasterFeedback>
  feedback_at_index  : int
  box_recorder       : optional<list<box>>
  static_count, dynamic_count : int

  # --- output -------------------------------------------------------------
  command_list : CommandList
```

Invariants:

- **`context_id` is the whole concurrency story.** Every visual carries an array of markers, one slot per context. A visual is "already collected" for *this* context when its slot holds this context's current marker. Two contexts collecting the same visual at the same time write different slots and never race. The cost is a fixed-size array on every visual in the level, and it is why the number of contexts is a compile-time constant.
- The marker is advanced exactly once per build, at the top. Nothing else may advance it — the shadow-caster cache in [`light_smapvis.cpp`](light_smapvis.cpp.md) exploits the fact that stamping `marker + 1` before the build pre-cancels a visual.
- Resetting a context clears every bucket and every scratch list but **does not** free their storage. A context is reused frame after frame and the buckets settle at the frame's working size; reallocating them would be the dominant cost of the build.
- The priority mask has two entries because a material's priority is halved to index it: materials are authored with a priority in a range twice as wide as the two groups that actually exist. Contexts that draw only one group — a depth prepass, for instance — set one entry.
- `sectors` and `portals` are per-context copies of the level's visibility topology. Each context walks and marks them independently, which is why the traversal marker lives in the sector rather than in the traverser.

## Exported units

- **`CasterFeedback`** (`R_feedback`) — the one-method hook the shadow-caster cache installs so the builder hands it the *n*-th static visual it accepts. See [`light_smapvis.cpp`](light_smapvis.cpp.md).
- **`RenderContext`** (`R_dsgraph_structure`) — everything above, plus:
  - `set_feedback`, `set_box_recorder`, the counter accessors and `reset` — bookkeeping, described by the invariants above. The box recorder captures the world-space bounding box of every visual accepted, which is how the deferred path builds a coarse structure for its light-volume culling.
  - `set_priority_mask` — which priority halves to admit, and whether decals are wanted.
  - `load` / `unload` — build this context's sectors and portals from the level's chunked data, wiring each portal to its two sectors and each sector to its portals and its root visual; and tear them down. Two passes are required because portals name sectors and sectors name portals.
  - `get_portal` / `get_sector` — index lookups.
  - `detect_sector` — which sector a point is in; see [`r__sector_detect.cpp`](r__sector_detect.cpp.md).
  - `add_static`, `add_leafs_dynamic`, `add_leafs_static`, `insert_dynamic`, `insert_static` — the collection funnel; see [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md).
  - `render_graph`, `render_hud`, `render_hud_ui`, `render_sorted`, `render_emissive`, `render_wmarks`, `render_distort`, `render_lods`, `render_box` — the drains; see [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) and [`r__dsgraph_render_lods.cpp`](r__dsgraph_render_lods.cpp.md).
  - `build_subspace` — the whole collection pass, top to bottom; see [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md).
