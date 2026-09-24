# src/Layers/xrRender/blenders/Blender_tree.cpp

> Trees and bushes: an alpha-tested surface whose vertex program bends it in the wind, with one flag that turns the wind off and repurposes the template as a distant-object impostor.

**Needs** — [`Blender_tree.h`](Blender_tree.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`Shader.h`](../Shader.h.md) · [`xrRender_console.h`](../xrRender_console.h.md)
**Used by** — reached through its declarations in [`Blender_tree.h`](Blender_tree.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

Foliage. The geometry is cards of leaves and branches with an alpha mask, and the animation is a vertex program that displaces each vertex by a wind function whose phase comes from the vertex's own position — so a forest of trees using one material all sway, out of phase, with no per-object state.

The second parameter is the one worth reading carefully. It is stored under the label "Object LOD" and it means **this material is not actually a tree**: it selects the still vertex program instead of the swaying one, and it drops the alpha reference to zero. It is how the distant-object impostor system reuses the foliage template for the flat billboards it substitutes for far-away objects, which must not sway and must not be cut out.

Forward renderer filling; the deferred renderers use [`Blender_tree_deferred.cpp`](Blender_tree_deferred.cpp.md).

## State

```text
RECORD Parameters
  alpha_blend : bool, default false
  not_a_tree  : bool, default false   # stored as "Object LOD"; added at version 1
```

**Invariants** — the alpha reference is **200 for a tree and 0 for an impostor**, derived from the flag rather than stored. 200 is the same hard cut-out reference the detail objects use, for the same reason: leaf cards need clean edges. An impostor is a pre-rendered image of a whole object and must keep its soft edges, so it tests at 0, which only discards fully transparent texels.

## `Save` / `Load`

**Contract** — the blend flag, then the impostor flag from version 1 on. A version-0 record leaves the impostor flag false, which is correct — the impostor system did not exist then.

## `Compile` — the fixed-function path

```text
FUNCTION compile_fixed(context)
  pass:
    depth test and write on
    blend := alpha_blend ? blend-by-alpha with a test at 200
                         : replace       with a test at 200
    vertex program := not_a_tree ? "tree_s" : "tree_wave"; no pixel program

    SELECT context.element
      normal_hq, normal_lq ->
        fog on, fixed lighting off
        stage 0: base texture DOUBLED against vertex colour, alpha from the texture
      lighting_only ->
        no lighting, no fog
        stage 0: vertex colour, alpha from the texture
```

**Notes** — The whole non-lightmapped fallback shape is present in the source and disabled, so this path runs unconditionally. The effect is that foliage always uses its vertex program, even in the configuration where every other template falls back to a fixed combine chain. That is defensible — foliage without the wind program looks wrong in a way a flat wall does not — but the fallback was written and then bypassed, and nothing records why.

Unlike the detail objects, the alpha reference here is the tree's literal 200 in both fixed-function shapes, ignoring the impostor flag. Only the programmable path derives it.

## `Compile` — the programmable path

**Contract** — one pass per element. The program names encode three independent facts: still or swaying, detailed or not, and which light is being added.

```text
FUNCTION compile_programmable(context)
  aref := not_a_tree ? 0 : 200
  blend := alpha_blend ? (src alpha : inv src alpha) : (one : zero)

  SELECT context.element
    normal_hq ->
      IF not_a_tree THEN
          programs (detail_diffuse ? "tree_s_dt"/"vert_dt" : "tree_s"/"vert")
      ELSE
          programs (detail_diffuse ? "tree_w_dt"/"vert_dt" : "tree_w"/"vert")
      fog on, depth test and write on, blend as above, alpha test at aref
      bind s_base <- instance texture 0
      bind s_detail <- the resolved detail texture

    normal_lq ->
      programs "tree_s" / "vert"        # the still program, always
      same state; bind s_base only

    add_point ->
      programs (not_a_tree ? "tree_s_point" : "tree_w_point") / "add_point"
      depth write off, blend one:one, alpha test at 0
      bind s_base, s_lmap and s_att <- point attenuation

    add_spot ->
      programs (not_a_tree ? "tree_s_spot" : "tree_w_spot") / "add_spot"
      depth write off, blend one:one, alpha test at 0
      bind s_base, s_lmap <- spot cookie with projective division, s_att <- attenuation

    lighting_only -> emit nothing
```

**Invariants**

- The low-quality world element always names the **still** program, even for a real tree. Wind is a high-quality-only feature, which is the same decision the detail objects make.
- The detail texture is bound **unconditionally** in the high-quality element, including in the branch whose programs do not sample it. That is harmless — a binding the compiled programs do not name is dropped — but it means the compiler must have resolved one, and the template declares itself detailable so that it will have.
- The light passes drop the alpha reference to 0 for trees as well as impostors. A leaf card lit by a point light is already masked by the base pass's depth; re-cutting it at 200 would create a second, slightly different silhouette and a rim of unlit texels.

**Notes** — The lighting-only element is empty and the source says why: lighting captured from foliage produced strange results. The consequence is that trees cast no lightmap contribution and no model shadow in the forward renderer.
