# src/Layers/xrRender/blenders/Blender_Blur.cpp

> A two-tap screen blur built out of fixed-function texture stages: two copies of the frame, each scaled by a constant, added together.

**Needs** — [`Blender_Blur.h`](Blender_Blur.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md)
**Used by** — reached through its declarations in [`Blender_Blur.h`](Blender_Blur.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description with no parameter block at all.

## Purpose

An internal template — not named by any shipped material, created by the engine directly — that blends two views of the same frame at half weight each. It is the primitive a multi-tap blur is built from: the caller supplies two textures offset from one another and draws a full-screen quad, and each invocation halves toward the average.

It is dead. Nothing in the current engine asks for it, in any renderer generation. It is kept because the class identifier is part of the frozen tag list and removing it would leave a hole; a rebuild may implement it as a stub, but must still answer to the tag.

## `Compile`

**Contract** — emits one pass, with no depth test, no depth write, no blending against the frame buffer, and no lighting or fog. Two texture stages and a shared constant colour.

```text
FUNCTION compile(context)
  pass:
    depth off, blend replace, lighting and fog off

    stage 0:  colour := texture0 * constant
              alpha  := texture0
              texture := instance texture 0, no matrix, no per-stage constant

    stage 1:  colour := texture1 * constant + running value
              alpha  := running value
              texture := instance texture 1, no matrix, uv channel 1

    the pass's shared constant colour := (127, 127, 127, 127)
```

**Notes** — The constant is 127 rather than 128 on all four channels, which is one step below half. On fixed-function hardware the multiply saturates at 255 and rounds toward zero, so two taps at 128 could exceed the original by a rounding step and brighten the image on every application; 127 guarantees it never does. That is the only decision in the file.

The two stages read different uv channels, which is how the caller supplies the offset: the offset is baked into the second channel of the quad's vertices, not into a matrix.
