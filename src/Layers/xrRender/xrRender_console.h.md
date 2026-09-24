# src/Layers/xrRender/xrRender_console.h

> The renderer's settings as seen by the code that reads them: one declaration per tuning variable, the flag-bit names, and two entry points.

**Needs** — [`xrRender_console.cpp`](xrRender_console.cpp.md)
**Used by** — [`Blender.cpp`](Blender.cpp.md) · [`Light_Render_Direct_ComputeXFS.cpp`](Light_Render_Direct_ComputeXFS.cpp.md) · [`R_Backend_Runtime.cpp`](R_Backend_Runtime.cpp.md) · [`SkeletonCustom.cpp`](SkeletonCustom.cpp.md) · [`SkeletonRigid.cpp`](SkeletonRigid.cpp.md) · [`Texture.cpp`](Texture.cpp.md) · [`BlenderDefault.cpp`](blenders/BlenderDefault.cpp.md) · [`Blender_BmmD.cpp`](blenders/Blender_BmmD.cpp.md) · [`Blender_Editor_Selection.cpp`](blenders/Blender_Editor_Selection.cpp.md) · [`Blender_Editor_Wire.cpp`](blenders/Blender_Editor_Wire.cpp.md) · [`Blender_LaEmB.cpp`](blenders/Blender_LaEmB.cpp.md) · [`Blender_Lm(EbB).cpp`](blenders/Blender_Lm(EbB).cpp.md) · [`Blender_Model.cpp`](blenders/Blender_Model.cpp.md) · [`Blender_Model_EbB.cpp`](blenders/Blender_Model_EbB.cpp.md) · _and 17 more_
**Tier floor** — T2: declarations of module-global storage, read directly at their use sites.

## Purpose

Declares the surface defined in [`xrRender_console.cpp`](xrRender_console.cpp.md), where
every variable's console name, clamp range, default and meaning lives. This header is what
a render pass includes in order to *read* a setting: the variables are globals, so a use
site names one and reads it, with no accessor and no indirection.

It is also where the bit names of the four flag words are defined — the renderer-common
word, the old renderer's word, and the main lighting word with its overflow companion.
Those names are how a pass asks "is tone mapping on"; their numeric values are internal.

## Exported units

- **The tuning variables** — roughly a hundred scalars, three vectors and four flag words,
  grouped by the renderer generation that introduced the name. The tables in the
  implementation twin are the authority on what each one does.
- **The token tables** — the accepted words for each token-valued variable (shadow-map
  size, sun quality, ambient-occlusion quality and mode, sun shafts, water reflection,
  multisampling, multisample alpha-test, min/max shadow maps). Each maps a shipped token
  text to the number stored behind it.
- **The flag-bit names** — one per console-exposed toggle, plus six with no console name.
- **The ambient-occlusion mode names** — off, classic, high-definition, horizon-based.
- **`xrRender_initconsole`** — register the whole set with the console, once, at renderer
  bring-up.
- **`xrRender_test_hw`** — declared here, implemented per backend: can this machine run
  this renderer.

**Notes** — One variable is declared here and defined in the engine rather than the
renderer: the supersampling factor, which the engine registers first and the renderer then
re-registers with a wider range. Nothing reads it. See the implementation twin.

Several variables carry a trailing comment giving a default that no longer matches the
value the implementation actually sets. Those comments record the *original* tuning from
before the project retuned the renderer; where they differ, the implementation is right.
