# src/Layers/xrRenderPC_GL/gl_rendertarget_u_set_rt.cpp

> Binding a phase's targets: the one operation every phase begins with, and the place the framebuffer's completeness is guaranteed.

**Needs** — [`gl_rendertarget.h`](gl_rendertarget.h.md) · [`../xrRenderGL/glR_Backend_Runtime.h`](../xrRenderGL/glR_Backend_Runtime.h.md) · [`../xrRenderGL/glHW.h`](../xrRenderGL/glHW.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`gl_rendertarget.h`](gl_rendertarget.h.md) · [`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md)
**Tier floor** — T1: it manipulates framebuffer attachments and asserts device-level completeness.

## Purpose

Every phase of the frame begins by saying which surfaces it writes. On an API with bindable output slots that is one call; here it is an attachment rewrite of the device's single framebuffer, plus a declaration of which attachments the pixel programs' outputs map to, plus a completeness check.

The decision worth extracting is the **dimension agreement rule**: a phase's target set defines one width and height, taken from the first target present, and every other target in the set is asserted to match. That invariant is what lets the rest of the renderer ask "how big is the current pass" without naming a surface, and it is why the phases never mix a full-resolution colour target with a quarter-resolution one.

## State

Writes the per-context pass dimensions declared in [`gl_rendertarget.h`](gl_rendertarget.h.md). Nothing else.

## `set_targets(command_list, colour_0, colour_1, colour_2, depth)`

**Contract** — binds up to three colour targets and a depth target for the phase about to run, on the device's one framebuffer. Records the pass dimensions from the first present target. Declares the draw-buffer mapping. Asserts that the framebuffer is complete and that every present target agrees on size. Any target may be absent, in which case its attachment point is cleared.

```text
FUNCTION set_targets(cmd, c0, c1, c2, depth) -> ()
  pass_width[cmd.context] := 0; pass_height[cmd.context] := 0
  draw_list := [none, none, none]

  cmd.set_framebuffer(device.framebuffer)      # always the same one

  FOR EACH (target, index) IN [(c0,0), (c1,1), (c2,2)]
    IF target EXISTS THEN
        IF pass dimensions already set THEN ASSERT target's size matches
        ELSE pass dimensions := target's size
        draw_list[index] := colour attachment `index`
        cmd.set_render_target(target.colour_handle, index)
    ELSE
        cmd.set_render_target(nothing, index)

  IF depth EXISTS THEN
      IF pass dimensions already set THEN ASSERT depth's size matches
      ELSE pass dimensions := depth's size
      cmd.set_depth_target(depth.depth_handle)
  ELSE
      cmd.set_depth_target(nothing)

  ASSERT pass dimensions are non-zero
  ASSERT the framebuffer is complete
  cmd.declare_draw_buffers(draw_list)
```

**Invariants** — the draw-buffer list is always declared with a full three entries even when fewer targets are bound, because the pixel programs' output locations are fixed at link time (three output names are bound unconditionally — see [`rgl_shaders.cpp`](rgl_shaders.cpp.md)). An output whose attachment is absent is discarded; an output whose *index* shifted would write to the wrong surface.

**Invariants** — the completeness check is not paranoia. This API refuses to draw to an incomplete framebuffer and *says nothing* — the draw is simply dropped. Every target-binding path in the renderer therefore checks, and a rebuild on this API must too, because the alternative is a phase that silently produces nothing.

**Notes** — the two-colour-target overload is the same procedure with the third target omitted, declaring only two draw buffers. Its size assertions mistakenly compare the *second* target's dimensions in the branch that should check the third; harmless in the three-target form only because no caller passes mismatched sizes. A rebuild with one parameterized implementation cannot make the mistake.

## `set_targets(command_list, width, height, colour_0, colour_1, colour_2, depth)` — the raw-handle form

**Contract** — the same binding against raw device handles and an explicitly given size, for the phases that write to the presentable surface rather than to a named target. Asserts the size is non-zero and the framebuffer complete; declares all three draw buffers.

**Notes** — this form exists because the presentable surface is reached through the device's back-buffer index rather than as a named target record. A rebuild that treats the swap-chain image as an ordinary target needs only the record-taking form.
