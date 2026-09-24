# src/Layers/xrRender/r__dsgraph_render_lods.cpp

> Drawing the distant world as imposters: each far object becomes one camera-facing quad blended between the two of its eight prebaked views that face the camera most nearly.

**Needs** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`FLOD.h`](FLOD.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md)
**Tier floor** — T1: it writes a fat interleaved vertex layout field by field into a mapped device buffer, in batches sized by how many fit at the buffer's stride.

## Purpose

Most of what a player sees at distance is trees and buildings that would cost thousands of triangles each. An *imposter* replaces one with four vertices. This file is the whole of how they are drawn, and its content is three decisions: the eight-view baking scheme, the blend between two views, and the batching.

## The imposter format

An imposter node holds **eight** prebaked faces, one per direction around the object — each a quad of four corners carrying a position, a texture coordinate, a baked sun contribution and a baked ambient-plus-hemisphere colour, plus the face's own normal. The eight are baked offline into the level data and are frozen.

Eight is the decision. Fewer and the switch between views is visible; more and the imposter atlas grows. The blend below is what makes eight sufficient.

The vertex written per imposter corner is unusually wide — **both** faces' positions, normals, texture coordinates and baked colours in one vertex, plus two blend factors — because the blend happens in the shader. That is why one imposter costs four vertices rather than eight, and why the batch size is computed from the stride.

## `render_lods`

**Contract** — Draw every queued imposter. Takes two flags: whether this is the depth-establishing pass (which also decides the direction and the material variant), and whether to empty the queue afterwards. Returns immediately when the queue is empty. Writes into the shared dynamic vertex stream in batches; issues one draw per material group per pass.

```text
FUNCTION render_lods(depth_pass, clear_after)
  IF queue is empty THEN RETURN

  # Direction: front-to-back for the depth pass (maximum rejection),
  # back-to-front otherwise (the imposters are alpha-blended against each
  # other during their cross-fade).
  items = depth_pass ? queue.front_to_back() : queue.back_to_front()

  element_slot = depth_pass ? lightmap_slot : low_quality_normal_slot

  # How many imposters fit in the shared vertex stream at once. Not a tuning
  # constant: it is the stream's capacity divided by four vertices at this
  # format's stride. The loop below therefore emits as many batches as the
  # stream forces.
  per_batch = vertex_stream_capacity / (stride * 4)

  FOR EACH batch OF at most per_batch items
    buffer = map_vertices(batch.count * 4, stride)
    current_material = first item's element ; run_length = 0

    FOR EACH item IN batch
      # Group consecutive items sharing a material into runs. The queue is
      # ordered by distance, so runs form naturally where a stand of the same
      # tree is at a similar depth; this is a cheap grouping, not a sort.
      IF item's element = current_material THEN run_length += 1
      ELSE  close the run ; current_material = item's element ; run_length = 1

      # --- Fade alpha ---------------------------------------------------
      # Linear across the band between the two imposter thresholds: fully
      # transparent where the real geometry still draws, fully opaque where
      # it no longer does. This is the other half of the cross-fade begun in
      # the collection pass.
      alpha = clamp(round((1 - (item.coverage - lod_exit) / (lod_enter - lod_exit)) * 255), 0, 255)

      # --- Choose and blend two of the eight faces -----------------------
      to_camera = normalise(item.centre - camera_position)
      rank the eight faces by dot(to_camera, face.normal)
      best, next, third = the top three dot products

      # The blend factor is NOT the ratio between the best two. It is derived
      # from where the second-best sits between the best and the THIRD:
      #   0.5 + 0.5 * (1 - (second - third) / (best - third))
      # which is 1 when the camera looks straight down the best face (second
      # and third are then equally bad) and 0.5 when it sits exactly between
      # two faces. Using the third as the baseline makes the factor depend on
      # the local spacing of the faces rather than on absolute dot products,
      # so the blend stays smooth as the camera circles the object.
      blend = round(clamp(0.5 + 0.5 * (1 - (next - third) / (best - third)), 0, 1) * 255)

      # --- Shift towards the camera -------------------------------------
      # Half the object's bounding radius along the view direction. An
      # imposter is a flat card standing where a solid object was; without the
      # shift its far half would poke out behind neighbouring geometry.
      shift = to_camera * -0.5 * item.bounding_radius

      # --- Emit four corners --------------------------------------------
      # In the order 3, 0, 2, 1 — the winding the shared quad index pattern
      # expects for this format. Each corner carries BOTH faces' data; the
      # shader interpolates between them using the blend factor, and the
      # alpha fades the whole card.
      FOR corner IN (3, 0, 2, 1)
        emit position, normal, texture coordinate and baked colour from the
          next-best face AND from the best face, both shifted,
          plus (baked sun of each, alpha, blend) packed into one colour
    close the final run
    unmap

    # --- Draw --------------------------------------------------------
    # Outer loop over pass slots, inner over material runs: pass zero of every
    # run, then pass one of every run, matching the opaque path's ordering.
    set world transform = identity
    FOR EACH pass slot
      offset = batch start
      FOR EACH run
        IF the run's element has a pass in this slot
          bind it and draw the run's quads as triangles
        advance offset by the run's four vertices per item
  IF clear_after THEN empty the queue
```

**Invariants**

- The corner order `3, 0, 2, 1` is fixed by the shared quad index pattern. Emitting them in natural order turns every imposter inside out.
- The batch size is derived from the stream's capacity, not chosen. A rebuild with a differently sized stream gets a different batch size and must not hard-code one.
- Every item in a batch uses the *first* item's geometry declaration and stride. This is safe only because every imposter in a level shares one vertex format — which is a property of the baked data, and worth stating because a rebuild that allowed several imposter formats would have to group by format as well as by material.
- The fade alpha and the blend factor are packed into a single colour together with the two faces' baked sun values. Four bytes, four meanings; the shader unpacks them, and the packing order is part of the shader contract.

**Notes** — The material element used differs between the two passes: the depth pass draws imposters with their lightmap element, the shading pass with a deliberately low-quality variant. Distant imposters get the cheapest shading in the material, which is invisible at the distance they are used and saves a meaningful fraction of the frame in a forest.
