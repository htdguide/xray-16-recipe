# src/Layers/xrRender/blenders/Blender_Editor_Wire.cpp

> The flat-coloured line material the authoring tools draw wireframes and gizmos with.

**Needs** — [`Blender_Editor_Wire.h`](Blender_Editor_Wire.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — reached through its declarations in [`Blender_Editor_Wire.h`](Blender_Editor_Wire.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The simplest template in the set: no texture, no lighting, no blending, no depth decision of its own — just vertex colour times a constant. Editor-only, and kept in the game build for the same reason as the selection highlight.

## State

```text
RECORD Parameters
  tint : text, default "$null"   # the name of a constant supplying the line colour
```

## `Save` / `Load`

**Contract** — one constant name, unversioned.

## `Compile`

```text
FUNCTION compile(context)
  IF lightmaps and dynamic lights are both off THEN
      one pass, whatever the default pipeline state is:
        stage 0: vertex colour * the pass constant, nothing bound
      RETURN

  one pass: programs "editor" / "simple_color", nothing else stated
```

**Notes** — Neither shape states a depth mode or a blend, so both inherit the pass recorder's defaults: depth test and write on, no blending. That is the right default for a wireframe drawn into a depth-buffered scene, and it means the file has nothing to say beyond "colour comes from the vertex and the constant".

The fixed-function shape opens a stage and never closes it before the pass ends. The recorder terminates the chain itself at pass end, so the result is the same one-stage pass; a rebuild need not reproduce the asymmetry.
