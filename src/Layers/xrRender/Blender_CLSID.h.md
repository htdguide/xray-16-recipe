# src/Layers/xrRender/Blender_CLSID.h

> The frozen set of material-template identifiers the shipped game data names.

**Needs** — _(none beyond the class-identifier packing helper in [`src/Common`](../../Common/README.md))_
**Used by** — [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) · [`BlenderDefault.cpp`](blenders/BlenderDefault.cpp.md) · [`Blender_Blur.cpp`](blenders/Blender_Blur.cpp.md) · [`Blender_BmmD.cpp`](blenders/Blender_BmmD.cpp.md) · [`Blender_BmmD_deferred.cpp`](blenders/Blender_BmmD_deferred.cpp.md) · [`Blender_Editor_Selection.cpp`](blenders/Blender_Editor_Selection.cpp.md) · [`Blender_Editor_Wire.cpp`](blenders/Blender_Editor_Wire.cpp.md) · [`Blender_LaEmB.cpp`](blenders/Blender_LaEmB.cpp.md) · [`Blender_Lm(EbB).cpp`](blenders/Blender_Lm(EbB).cpp.md) · [`Blender_Model.cpp`](blenders/Blender_Model.cpp.md) · [`Blender_Model_EbB.cpp`](blenders/Blender_Model_EbB.cpp.md) · [`Blender_Model_EbB_deferred.cpp`](blenders/Blender_Model_EbB_deferred.cpp.md) · [`Blender_Particle.cpp`](blenders/Blender_Particle.cpp.md) · [`Blender_Particle_deferred.cpp`](blenders/Blender_Particle_deferred.cpp.md) · _and 15 more_
**Tier floor** — T1: each value is eight ASCII bytes reinterpreted as one 64-bit integer, and the byte order of that reinterpretation is what the shipped data encodes.

## Purpose

Every material in the shipped material library names its template by an eight-character tag, padded with spaces to exactly eight. This file is the list of tags the engine answers to. **It is frozen**: a level whose material references `"LmBmmD  "` will not load if the rebuild spells it differently or pads it differently.

## State

```text
CONSTANTS  # tag, padded to exactly 8 characters with trailing spaces

# level surfaces
B_DEFAULT       "LM      "   # lightmapped diffuse — the workhorse for world geometry
B_DEFAULT_AREF  "LM_AREF "   # ditto, with alpha testing (foliage, grates)
B_VERT          "V       "   # vertex-lit diffuse, for geometry with no lightmap
B_VERT_AREF     "V_AREF  "
B_LmBmmD        "LmBmmD  "   # lightmapped + bump, with the detail-bump convention
B_LaEmB         "LaEmB   "   # lightmap-alpha, emissive, bump
B_LmEbB         "LmEbB   "   # lightmap, emissive-by-bump
B_B             "BmmD    "   # bump, no lightmap
B_BmmD          "BmmDold "   # the superseded spelling of the same idea, still present in data

B_PARTICLE      "PARTICLE"

# screen-space / effect
B_SCREEN_SET    "S_SET   "   # blit, replace
B_SCREEN_GRAY   "S_GRAY  "   # blit, desaturate
B_LIGHT         "LIGHT   "
B_BLUR          "BLUR    "
B_SHADOW_TEX    "SH_TEX  "   # project a shadow texture
B_SHADOW_WORLD  "SH_WORLD"   # receive a projected shadow

# procedurally placed geometry
B_DETAIL        "D_STILL "   # grass/debris, wind-animated
B_TREE          "D_TREE  "   # tree, wind-animated by a different wave model

# dynamic models
B_MODEL         "MODEL   "
B_MODEL_EbB     "MODELEbB"   # model with emissive-by-bump

# authoring tools only
B_EDITOR_WIRE   "E_WIRE  "
B_EDITOR_SEL    "E_SEL   "
```

**Invariants**

- Exactly eight characters, space-padded. The packing helper reads eight bytes; a shorter literal reads past its own terminator, and a longer one truncates. Both produce a tag that matches nothing.
- The mapping from tag to template class is **per backend** (see [`Blender.cpp`](Blender.cpp.md)). This list is the shared vocabulary; not every backend answers every tag, and the two editor tags are absent from the game backends entirely.

## Notes

The names are abbreviations of the layer stack the template builds, in the order the layers apply: `L` lightmap, `a` alpha, `m` modulate, `B`/`b` bump, `D` detail, `E`/`Eb` emissive (by bump). Reading them that way is the only documentation that exists for what each template does, and it is worth reading them that way — `LmBmmD` is *lightmap, modulate, bump, modulate-2x, detail*.

`"BmmDold "` is the one genuine puzzle: a template kept alive under a renamed tag so that data authored against the older spelling still loads, while `"BmmD    "` was reused for a different stack. Which of the two the art actually uses varies by game generation. A rebuild must implement both and must not merge them.
