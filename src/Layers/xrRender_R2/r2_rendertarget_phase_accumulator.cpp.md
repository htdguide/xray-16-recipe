# src/Layers/xrRender_R2/r2_rendertarget_phase_accumulator.cpp

> Binds the lighting accumulator, clearing it the first time each frame, and does the same
> for the separate volumetric accumulator.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md)
**Used by** — [`r4_rendertarget_accum_direct.cpp`](../xrRenderPC_R4/r4_rendertarget_accum_direct.cpp.md) · [`r2_rendertarget_accum_reflected.cpp`](r2_rendertarget_accum_reflected.cpp.md) · [`r3_rendertarget_accum_point.cpp`](r3_rendertarget_accum_point.cpp.md) · [`r3_rendertarget_accum_spot.cpp`](r3_rendertarget_accum_spot.cpp.md)
**Tier floor** — T1: target binding, clears and stencil state.

## Purpose

Every light, the sun, the emissive pass and the indirect bounces all add into one
accumulator, and each of them calls this before it draws. It must therefore be cheap to
call repeatedly and must clear exactly once per frame. It is also where the viewport is
restored, because the pass that ran before an accumulation is usually a shadow pass that
shrank the viewport to an atlas rectangle.

## `phase_accumulator`

**Contract** — binds the accumulator with the scene depth, sets stencil to admit only
pixels the G-buffer covered, disables culling, enables colour writes, restores the full
viewport. On the frame's first call it also clears the accumulator to black and resets the
light marker. Idempotent within a frame.

```text
FUNCTION phase_accumulator()
  IF the accumulator was already cleared this frame
      bind the accumulator (or its blend twin, on devices without float blending)
          with the scene depth
  ELSE
      record this frame as cleared
      bind the accumulator with the scene depth
      reset the light marker
      clear the accumulator to black
      stencil: pass where the stored value is at least 1
      cull nothing; colour writes on
  restore the full viewport
```

**Invariants** — the clear-once marker is a frame number, not a flag, so it self-resets
across frames with no teardown. The first call of a frame is the sun's (or, with no sun,
the first light's); nothing may read the accumulator before it.

**Notes** — the blend twin is bound instead of the accumulator on devices that cannot
blend into a float target: those accumulate into the twin and copy across afterwards, and
the copy target is the accumulator itself, so the twin must be what is bound during the
draw. Note that the *clear* still targets the real accumulator — the twin is transient per
light and is cleared by its own blend-copy.

Culling is disabled here and re-enabled by each caller, because the volume-marking steps
need front and back faces separately and the full-screen steps need neither.

## `phase_volumetric_accumulator`

**Contract** — the same for the separate target light shafts accumulate into: binds it
with the scene depth, clears it to black on first touch of the frame, disables stencil and
culling.

**Notes** — light shafts get their own target rather than sharing the accumulator because
they are composited differently: the accumulator is multiplied by albedo at combine time,
while shafts are additive and must not be tinted by the surface they are seen against. The
"first touch" flag here is a boolean reset at scene preparation rather than a frame
number, because unlike the accumulator this target may legitimately go a whole frame
without being touched.
