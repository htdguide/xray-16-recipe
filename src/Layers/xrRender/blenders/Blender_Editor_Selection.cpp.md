# src/Layers/xrRender/blenders/Blender_Editor_Selection.cpp

> The translucent highlight the authoring tools draw over a selected object.

**Needs** — [`Blender_Editor_Selection.h`](Blender_Editor_Selection.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — reached through its declarations in [`Blender_Editor_Selection.h`](Blender_Editor_Selection.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

An editor-only template. A rebuild that ships only the game may omit it entirely — but the class tag stays in the frozen list, and the *game* engine still compiles the template, because the editor and the game share one material library and one identifier table.

## State

```text
RECORD Parameters
  tint : text, default "$null"   # the name of a constant supplying the highlight colour
```

## `Save` / `Load`

**Contract** — one constant name, unversioned.

## `Compile`

**Contract** — two shapes, chosen by the same lighting-configuration test every forward template uses.

```text
FUNCTION compile(context)
  IF lightmaps and dynamic lights are both off THEN     # the tools' own configuration
      one pass:
        depth test ON, depth write OFF, blend by source alpha
        stage 0: the pass constant * vertex colour, clamped addressing,
                 the template's base texture and matrix bound but unsampled
      RETURN

  one pass:
    programs "editor" / "simple_color"
    fog on, depth test on, depth write off, blend by source alpha
    no textures at all
```

**Invariants** — depth testing is on and depth writing is off in both shapes. The highlight must be occluded by geometry in front of it (so a selected object behind a wall does not glow through) and must not occlude anything itself.

**Notes** — Declares itself **not lightmappable**, explicitly rather than by inheritance, which keeps the level compiler from allocating lightmap area for editor gizmos that end up in a level's geometry during authoring.
