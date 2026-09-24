# src/Layers/xrRender_R2/r2_R_render.cpp

> The frame graph: every pass of a deferred frame, in order, with the joins where
> background visibility work is collected.

**Needs** — [`r2.h`](r2.h.md) · [`r2_types.h`](r2_types.h.md) ·
[`r2_R_lights.cpp`](r2_R_lights.cpp.md) ·
[`r3_rendertarget_phase_scene.cpp`](r3_rendertarget_phase_scene.cpp.md) ·
[`r3_rendertarget_phase_occq.cpp`](r3_rendertarget_phase_occq.cpp.md) ·
[`r3_rendertarget_mark_msaa_edges.cpp`](r3_rendertarget_mark_msaa_edges.cpp.md) ·
[`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) ·
[`xrRender/r__dsgraph_structure.h`](../xrRender/r__dsgraph_structure.h.md) ·
[`xrRender/light.h`](../xrRender/light.h.md) ·
[Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it sequences device state changes and target binds with an ordering
that is itself the correctness condition.

## Purpose

This is the page the rest of the chapter exists to serve. It states what is drawn into
what, when, and what each step reads. Nothing here computes anything — every step is a
call into a pass that has its own twin — but the *order* is a decision, and almost every
adjacency in it is load-bearing.

`render_menu` lives here too, because it is the same frame graph with everything but the
last two steps removed.

## State

Stateless — it reads the renderer's state and drives it.

## `render`

**Contract** — draws one frame of the loaded level into the swap chain's colour buffer.
Blocks once, on the previous frame's fence. Does nothing but bind the back buffer if no
level is loaded or the menu is covering the scene, and skips entirely on the first frame
after a device reset. Leaves the immediate context with the back buffer bound and the
distortion list empty.

**Invariants** — every phase started in [`r2_R_calculate.cpp`](r2_R_calculate.cpp.md) is
joined exactly once on every path through this function. The accumulator is cleared by
whichever lighting step touches it first, not by an explicit step. Stencil bit meanings
hold throughout: bit 0 set means "a surface was written here", the high bit set means
"this pixel's samples disagree".

```text
FUNCTION render()
  set the normal viewport
  IF no level OR the menu hides the scene
      bind the back buffer; RETURN
  IF first frame after reset
      clear the flag; RETURN

  # ---- 1. optional depth prefill -------------------------------------------------
  IF depth_prefill_enabled
      build a projection with the same field of view but a near far plane
          (a fraction of the weather's far plane)
      walk the scene graph from the camera's sector with that transform,
          priority 0 only, occluders on, precise portals on
      # drawn below, after the main walk is joined, so the two walks do not interleave

  # ---- 2. join the frame's start --------------------------------------------------
  AWAIT previous frame's fence                       # bounds how far ahead the CPU runs
  AWAIT main visibility phase
  prepare the scene targets (clear policy, below)
  IF depth_prefill_enabled
      draw the prefill set with colour writes off    # writes depth only

  # ---- 3. G-buffer fill -----------------------------------------------------------
  bind position + normal + albedo + depth; stencil writes 1 on every covered pixel
  draw the heads-up display                          # its own compressed depth range
  draw the scene graph, priority 0
  draw the level-of-detail impostors
  draw the detail (grass) layer
  end the scene pass                                 # resolves albedo if the workaround is on

  # ---- 4. occlusion-test the light volumes ----------------------------------------
  bind depth only, colour writes off, stencil >= 1
  FOR EACH light in the database (point, spot, shadowed, round-robin across the three)
      draw its bounding volume as an occlusion query
      IF the answer is already available THEN put it in the answered package
      ELSE put it in the pending package
  sort both packages                                 # by shadow need, then by size

  # ---- 5. decals -------------------------------------------------------------------
  IF the active item wants its own overlay
      bind albedo; draw the first-person item's overlay
  bind albedo; draw wallmarks                        # priority: they are normal geometry

  # ---- 6. retire last frame's queries ---------------------------------------------
  FOR EACH light carried over from the previous frame
      collect its shadow-visibility query results
  clear the carried-over list

  # ---- 7. multisample edge marking -------------------------------------------------
  IF multisampling
      full-screen pass setting the high stencil bit where samples disagree

  # ---- 8. rain wetness --------------------------------------------------------------
  AWAIT rain phase                                   # rewrites normal, albedo, gloss

  # ---- 9. the sun --------------------------------------------------------------------
  AWAIT sun phase (cascaded or legacy)               # each cascade already accumulated
  blend the accumulated sun

  # ---- 10. emissive ---------------------------------------------------------------
  bind the accumulator; restore the camera transforms
  stencil: always write 1
  draw the self-illuminated set

  # ---- 11. lights -------------------------------------------------------------------
  accumulate the answered package
  accumulate the pending package

  # ---- 12. combine and post ---------------------------------------------------------
  combine                                            # backend-owned; see chapter 20/21
```

### Why this order

**The prefill is separate from the main walk.** Both walk the same scene graph, so they
cannot share a command context; the prefill is issued through the immediate context
before the main phase is joined, and the main phase's *calculate* is forced onto the
calling thread whenever the prefill is enabled. That is the whole reason the option
disables parallel visibility.

**Occlusion queries come after the G-buffer and before any lighting.** They need the
frame's finished depth to be meaningful, and their answers need a gap before they are
read or the wait costs more than the culling saves. The gap is filled with decals, query
retirement and edge marking — work that does not depend on the answers. Splitting the
lights into *answered* and *pending* and accumulating the answered set first widens the
gap further for the ones still in flight.

**Rain runs before the sun.** Wetness rewrites the normal and the gloss of surfaces the
sky can see; every subsequent lighting pass must see the wet values, and the sun is the
first of them.

**The sun runs before the local lights.** The sun's own cascade passes each end by adding
into the accumulator, and the accumulator's first touch is what clears it. Ordering the
sun first means the clear happens once, before anything else is in there. It is also why
emissive geometry is drawn *after* the sun rather than at G-buffer time, where it would
seem to belong: it writes directly into the accumulator and would otherwise be clobbered
by the clear.

**The wallmark pass is bound to albedo, not to the frame.** Decals are part of the
surface, so they must participate in lighting; drawing them into albedo before any light
runs is what makes a bullet hole receive shadows.

### The scene split

**Contract** — an option splits the G-buffer fill in two, drawing the static scene graph
first, then taking the occlusion-query detour, then returning for the heads-up display,
the impostors and the detail layer. The same set is drawn either way.

**Notes** — the split exists to give the occlusion queries their latency gap *inside* the
G-buffer pass rather than after it, on hardware where the second half of the fill was
long enough to cover the query. It changes no output. A rebuild can drop it and keep the
unsplit path; the twin records it because the two branches are visibly different code and
a reader will ask which one is canonical. Neither is: the option is a tuning switch.

## `render_forward`

**Contract** — draws everything deferred shading cannot express, after the deferred
resolve has produced a low-dynamic-range image. Called from the combine step, not from
`render`. Clears the level-of-detail cache first so that impostors chosen this frame do
not leak into the next.

```text
FUNCTION render_forward()
  drop the level-of-detail cache
  draw the scene graph, priority 1        # translucent world geometry
  draw faded portals                      # the portal itself, alpha-faded
  draw the strictly sorted set            # back-to-front: glass, water surfaces, particles
  draw the weather's last layer           # rain streaks, lightning
```

**Notes** — priority 1 is the second of the two priority classes a material can declare.
Priority 0 is everything that writes the G-buffer; priority 1 is everything that must be
composited afterwards. Glass and water are ordinary members of the sorted set — they are
not special-cased anywhere in this chapter, they are materials whose pass description
happens to request sorting and blending. Water's screen-space reflection and refraction
are entirely a property of its shipped program plus the two engine-supplied constants
(water intensity, and the scene image bound as a texture), which is why no "water pass"
appears in the frame graph.

## `render_menu`

**Contract** — draws the main-menu screen: the user-interface layer into one
low-dynamic-range target, the distortion mask into a second, then one full-screen quad
that combines them into the back buffer. Does not touch the G-buffer, the accumulator or
any light.

```text
FUNCTION render_menu()
  bind generic0 + the back depth; draw the menu's UI layer
  bind generic1 + the back depth; clear it to "no displacement"; draw the UI's
      distortion layer
  bind the back buffer; draw one full-screen quad with the distortion material
```

**Notes** — the neutral distortion value is mid-grey with mid alpha, because the mask
encodes a signed two-dimensional offset in two channels around the midpoint. A clear to
black would displace the whole screen.

## `before_world_render` / `after_world_render`

**Contract** — empty hooks, called around the world render by the game layer so that a
modification can insert work without editing the frame graph. Recorded because their
existence is part of the extension surface.
