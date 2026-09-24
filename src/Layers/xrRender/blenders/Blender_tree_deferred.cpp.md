# src/Layers/xrRender/blenders/Blender_tree_deferred.cpp

> The foliage template filled for the deferred renderers: a g-buffer write with an optional alpha-to-coverage prepass, and a shadow-map element with its own four vertex programs.

**Needs** — [`Blender_tree.h`](Blender_tree.h.md) · [`uber_deffer.h`](uber_deffer.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

The deferred renderers answer the foliage class tag with this file instead of [`Blender_tree.cpp`](Blender_tree.cpp.md); exactly one is built, and everything but `Compile` is duplicated verbatim.

The impostor flag keeps its meaning and gains a second consequence: it selects not only the world-pass vertex program but a *different shadow-map vertex program*, because a shadow caster must be displaced by the same wind function as the surface it shadows or the shadow will detach from the tree.

## The shadow program naming

**This is the part a rebuild must reproduce exactly.** Four names, chosen by two flags:

```text
             not blended                  blended (alpha-tested in the shadow pass)
tree         "shadow_direct_tree"         "shadow_direct_tree_aref"
impostor     "shadow_direct_tree_s"       "shadow_direct_tree_s_aref"
```

The `_s` means still, the `_aref` means the program also has to run the alpha cut-out. They are shipped shader sources and the spelling is frozen.

## `Compile`

```text
FUNCTION compile(context)
  world_vertex_program  := not_a_tree ? "tree_s" : "tree"
  shadow_vertex_program := per the table above
  atoc := alpha_blend AND the device resolves alpha tests through coverage

  SELECT context.element
    normal_hq, normal_lq ->
      IF atoc THEN
          shared deferred emission, pixel program "base_atoc"
          mark the stencil, disable colour writes, enable alpha-to-coverage
          end the pass
      shared deferred emission, pixel program "base", alpha-tested per alpha_blend
      mark the stencil
      IF atoc THEN require depth EQUAL     # confine to the samples coverage accepted
      end the pass

    shadow ->
      IF alpha_blend THEN
          programs shadow_vertex_program / "shadow_direct_base_aref"
          blend zero : one, alpha test at 200
      ELSE
          programs shadow_vertex_program / the do-nothing pixel program
      bind s_base <- instance texture 0 through the linear sampler
      disable colour writes
```

**Invariants**

- The shadow pass writes no colour. A shadow map holds depth; where the device can compare depth in the sampler, the pixel program is the do-nothing one and the whole pass is a depth-only draw.
- The alpha-tested shadow variant still tests at the literal **200**, matching the world pass's tree reference. A shadow cut at a different reference than the surface produces a shadow that does not match the leaf.
- The world elements mark the stencil with the g-buffer value under the high-bit-preserving mask, like every g-buffer-writing template.

**Notes** — The oldest deferred renderer has no coverage path, no stencil marking, and only two shadow program names (it derives `_aref` from the pixel program instead of from the vertex one). The extra two names arrived with the generation that made the shadow pass depth-only, because a depth-only pass has no pixel program to put the cut-out in.
