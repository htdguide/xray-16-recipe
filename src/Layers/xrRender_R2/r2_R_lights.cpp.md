# src/Layers/xrRender_R2/r2_R_lights.cpp

> Packs shadow-casting lights into atlas batches, renders each batch's shadow maps, and
> accumulates every light in an order that keeps the atlas hot and the stencil valid.

**Needs** — [`r2.h`](r2.h.md) · [`SMAP_Allocator.h`](SMAP_Allocator.h.md) ·
[`r2_rendertarget_phase_smap_S.cpp`](r2_rendertarget_phase_smap_S.cpp.md) ·
[`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) ·
[`r3_rendertarget_accum_point.cpp`](r3_rendertarget_accum_point.cpp.md) ·
[`r3_rendertarget_accum_spot.cpp`](r3_rendertarget_accum_spot.cpp.md) ·
[`r2_rendertarget_accum_reflected.cpp`](r2_rendertarget_accum_reflected.cpp.md) ·
[`xrRender/light.h`](../xrRender/light.h.md) ·
[`xrRender/Light_Package.h`](../xrRender/Light_Package.h.md) ·
[`xrRender/Light_Render_Direct.h`](../xrRender/Light_Render_Direct.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it interleaves command-context allocation, background visibility
tasks and device queries whose completion order is the correctness condition.

## Purpose

One shadow atlas, many shadow-casting lights. This file decides how they share it, and in
what order everything is added into the accumulator. It is the busiest scheduling code in
the renderer: for each batch it fans out a visibility walk per light onto worker threads,
collects them in order, renders their shadow maps into disjoint atlas rectangles, and then
drains the batch into the accumulator — while also interleaving the unshadowed lights,
which need no atlas at all and are therefore free work to hide the latency behind.

## State

Stateless between calls; the packages it consumes and the atlas allocator it drives belong
to the renderer.

## `render_lights`

**Contract** — consumes a light package entirely: after it returns, the package's point,
spot and shadowed lists are empty and every visible light in them has been added to the
accumulator. Allocates and releases command contexts from the shared pool; blocks on
worker tasks and, indirectly, on the device. Called twice per frame — once for the lights
whose occlusion answers were ready, once for those that were still pending.

**Invariants** — a light that failed its visibility update is dropped, not drawn. Every
light that enters the shadowed list leaves with an atlas rectangle and a batch identifier.
Every command context allocated is released. The accumulator must be bound before any
accumulation and is *not* bound during shadow rendering — the two alternate, and the pass
that re-binds the accumulator also restores the viewport the shadow pass shrank.

```text
FUNCTION render_lights(package)
  # ---- 1. drop the invisible, compute each shadowed light's projection ------------
  FOR EACH light IN package.shadowed
      update its visibility
      IF not visible THEN remove it
      ELSE compute its spot view and projection, and the atlas side it deserves

  # ---- 2. assign every shadowed light to a batch ----------------------------------
  batch_id = 0
  WHILE some light is unassigned
      reset the atlas allocator
      sort the unassigned by descending requested side
      FOR EACH light IN that order
          IF the allocator can place it
              record its rectangle and this batch_id on the light
              move it to the assigned list
      batch_id = batch_id + 1
  reverse the assigned list                # the drain loop pops from the back

  # ---- 3. drain, one batch at a time ----------------------------------------------
  WHILE some shadowed light remains
      clear the atlas
      current = the batch identifier of the last light
      queue = empty
      WHILE the last light still belongs to `current`
          context = allocate a command context
          IF none is free
              flush(queue)                 # frees contexts; then retry
              CONTINUE
          pop the light; remember it for next frame's query retirement
          start its shadow-visibility query
          SPAWN (or run inline): walk the scene graph from the light's sector,
              phase = shadow, priority 0 (and 1 if translucent shadows are on),
              view position = the light, transform = the light's combined matrix,
              frustum = the light's frustum *without its near plane*
          append (light, task, context) to queue
      flush(queue)

      # ---- 4. accumulate, with the unshadowed lights interleaved -----------------
      bind the accumulator
      IF an unshadowed point light remains: pop one, accumulate it, then its bounces
      IF an unshadowed spot light remains:  pop one, accumulate it, then its bounces
      FOR EACH light whose shadow map is in this batch
          accumulate it, then its bounces
      IF volumetric lights are enabled
          FOR EACH light in this batch: accumulate its light shafts
      clear the batch's light list

  # ---- 5. anything left over --------------------------------------------------------
  accumulate every remaining unshadowed point light, then every remaining spot light
```

### `flush` — render one batch's shadow maps

```text
FUNCTION flush(queue)
  FOR EACH (light, task, context) IN queue        # in the order they were queued
      AWAIT task
      opaque      = the context's priority-0 sets are non-empty
      translucent = the context's priority-1 or sorted sets are non-empty
      IF opaque OR translucent
          bind the atlas with this light's rectangle as the viewport
          set world = identity, view = the light's view, projection = the light's
          draw priority 0
          IF the sun-details option is on, draw the detail layer too
          IF translucent
              re-bind for the coloured-mask pass; draw priority 1; draw the sorted set
      ELSE
          count it as "clipped": the light casts no shadow this frame
      end the light's shadow-visibility query
      release the context
  empty the queue
```

**Notes on the batching.** The sort is by *descending* requested side, and it is redone
before every batch rather than once. That matters: after a batch fills, the lights left
over are not a suffix of the original order — the allocator may have skipped a large light
and taken two small ones — so re-sorting the remainder is what keeps the next batch's
first-fit good. The reversal at the end is not cosmetic; the drain loop pops from the back
so that lights of one batch are contiguous, and reversing makes the pop order match the
placement order, which is the order the atlas rectangles were laid out in.

**Notes on the light frustum.** The shadow walk builds the light's frustum from its
combined matrix with every plane *except the near plane*. A caster between the light and
its near plane still shadows, so clipping it away would punch holes in the shadow.

**Notes on the interleave.** One unshadowed point and one unshadowed spot are accumulated
per batch, not all of them. This is deliberate rationing: the unshadowed lights are the
cheap work available to fill the gap between submitting a batch's shadow draws and needing
their results, and spending them one per batch spreads them across every gap instead of
burning them all in the first one. Whatever is left when the shadowed lights run out is
drained at the end.

**Notes on context exhaustion.** When the pool has no free context the loop flushes what
it has and retries rather than blocking. That bounds the number of shadow walks in flight
to the pool size and is what makes the pool size a real tuning knob.

**Notes on the query lifetime.** A light's shadow-visibility query is started before its
walk and ended after its draws, but its *result* is not read until the next frame — the
lights are pushed onto a carry-over list that [`r2_R_render.cpp`](r2_R_render.cpp.md)
drains at a fixed point. Reading in the same frame would stall. The consequence is that
shadow-visibility culling is always one frame stale, which is invisible in practice and is
the standard bargain for hardware occlusion.

## `render_indirect`

**Contract** — for one light, accumulates its precomputed indirect bounces as a set of
synthetic spot lights. Does nothing unless global illumination is enabled or the light has
no bounce list. Allocates one synthetic light on the stack and reuses it for every bounce.

```text
FUNCTION render_indirect(light)
  IF global illumination is off OR light.bounces IS EMPTY THEN RETURN
  bounce_light = a reflected-type light, no shadow, full hemisphere cone
  source_energy = intensity of the light's colour
  FOR EACH bounce IN light.bounces
      energy = source_energy * bounce.energy
      IF energy < clip_threshold THEN CONTINUE        # too dim to be worth a draw
      bounce_light.colour = light.colour scaled by bounce.energy
      orient bounce_light along the bounce direction, with any perpendicular as "up"
      place it at the bounce point, in the bounce's sector
      # range from a linear falloff: E_max / (1 + x) = E_min, solved for x
      range = (energy - one_255th) / one_255th
      IF range < 0.1 THEN CONTINUE
      accumulate bounce_light as a reflected light
```

**Notes** — the range derivation is the interesting line. A bounce has an energy but no
natural radius; inverse-square would give it infinite reach. The code instead assumes a
*linear* falloff and solves for the distance at which the bounce drops to one part in 255
— the smallest value an eight-bit frame can show — so the volume is exactly as large as
can matter and no larger. The one-tenth floor discards bounces whose volume would be
smaller than the cost of drawing it.

The bounce list itself is precomputed and lives on the light; this file only replays it.
Reflected lights are full-hemisphere cones with no shadow, which is why they can be drawn
as cheap sphere volumes with a direction-weighted term rather than as real spot lights.
