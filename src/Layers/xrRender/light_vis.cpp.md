# src/Layers/xrRender/light_vis.cpp

> Decides whether a light is worth drawing this frame, by drawing its volume into an occlusion query — but not every frame, and not when the camera is inside it.

**Needs** — [`light.h`](light.h.md) · [`r__occlusion.h`](r__occlusion.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it issues an occlusion query around a volume draw and reads the fragment count back, with the latency that implies.

## Purpose

A light that is entirely behind geometry still costs a full-screen-ish volume draw and, if it shadows, a whole shadow map. Testing it with an occlusion query is much cheaper — but a query costs a round trip to the device, so testing every light every frame would replace one cost with another.

This file is the scheduling policy around that test. Its content is three decisions: when a test may be skipped outright, how long an answer is trusted, and what "visible" means in fragments.

## Scheduling constants

```text
retest_delay_when_trivially_visible : 1 to 3 frames    (uniform)
retest_delay_when_visible           : 10 to 20 frames  (uniform)
retest_delay_when_invisible         : 1 frame
visible_fragment_threshold          : 4
```

The asymmetry is the decision. A light found **visible** is retested rarely, because a visible light that becomes hidden merely costs a few wasted frames of drawing. A light found **invisible** is retested on the very next frame, because a hidden light that becomes visible and is not noticed *pops on*, which the eye catches immediately. The randomised interval spreads the tests of many lights across frames instead of clustering them.

The fragment threshold is not zero: a light peeking through a crack four pixels wide is treated as hidden. Four is an authored constant with no derivation.

## `prepare_visibility`

**Contract** — Possibly issue an occlusion test for this light. Called once per frame per candidate light, before the light is drawn. Writes into this light's visibility record and, when it tests, into the command list's state and the occlusion query pool. Returns nothing; the answer arrives in `update_visibility`.

```text
FUNCTION prepare_visibility(command_list)
  # Regeneration hook: the indirect photon budget is a runtime setting, so a
  # change to it must rebuild the bounce set. This is the only place that
  # notices, because it is the only per-frame per-light entry point.
  IF indirect_photon_count differs from the configured budget
    generate_indirect()

  IF current_frame < next_test_frame THEN RETURN    # a scheduled answer still stands

  # --- Skip conditions ---------------------------------------------------
  # 1. Configured opt-outs, separately for shadowed and unshadowed lights.
  #    Unshadowed lights are cheap enough that testing them is usually a loss,
  #    and the shipped default opts them out.
  skip = (unshadowed and unshadowed tests disabled)
      or (shadowed   and shadowed   tests disabled)

  # 2. The camera is inside, or nearly inside, the light's volume. A volume
  #    that encloses the near plane cannot be tested: it would be clipped away
  #    and read as invisible. The margin is the radius of the sphere that
  #    circumscribes the near plane rectangle, derived from the near distance
  #    and the two half-angles of the frustum — so it is exact for the current
  #    field of view and aspect, not a guess.
  near_margin = circumscribed_radius_of_near_plane()
  IF skip OR camera_position.distance_to(sphere.centre) <= sphere.radius * 1.01 + near_margin
    visible        = true
    pending        = false
    next_test_frame = current_frame + random(1, 3)
    RETURN

  # --- Issue the test ----------------------------------------------------
  pending = true
  rebuild_volume_transform()
  command_list.set_world_transform(volume_transform)
  query_order = begin_occlusion_query(query_id)

  # Stencil: normally the volume is tested only where the stencil marks real
  # geometry, which keeps the sky from counting as "the light is visible".
  # A shadowed VOLUMETRIC spot is the exception: its god-rays are visible
  # wherever its frustum is, including against the sky, so its test runs with
  # the stencil off.
  IF spot AND shadowed AND volumetric
    command_list.disable_stencil_test()
  ELSE
    command_list.enable_stencil_test(pass where stencil <= 1)

  draw_light_volume(command_list, this)
  end_occlusion_query(query_id)
```

**Invariants** — The sphere radius is inflated by one percent before the inside test. That margin absorbs the difference between the conservative enclosing sphere and the volume actually drawn; without it a camera just outside the sphere but inside the cone would test a clipped volume and blink the light off.

## `update_visibility`

**Contract** — Read the result of a test issued earlier this frame and reschedule. Blocks on the query result. Does nothing when no test is pending.

```text
FUNCTION update_visibility()
  IF NOT pending THEN RETURN
  fragments = read_occlusion_query(query_id)
  visible   = fragments > visible_fragment_threshold
  pending   = false
  next_test_frame = current_frame + (visible ? random(10, 20) : 1)
```

**Notes** — Query results are read in the same frame the query was issued, which is the expensive way to use occlusion queries: the read stalls until the device has finished. The engine accepts the stall because it batches all lights' queries before reading any of them, and because the alternative — a one-frame-late answer — makes lights lag visibly behind the camera. The shadow-caster cache in [`light_smapvis.cpp`](light_smapvis.cpp.md) makes the opposite choice for the opposite reason; the two are worth reading together.
