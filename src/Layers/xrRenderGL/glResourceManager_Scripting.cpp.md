# src/Layers/xrRenderGL/glResourceManager_Scripting.cpp

> Exposes the material compiler to the shipped shader scripts: the exact vocabulary a `.s` file may use, and the order a material's five level-of-detail elements are compiled in.

**Needs** — [`Blender_Recorder_GL.cpp`](Blender_Recorder_GL.cpp.md) · [`xrRender/Blender_Recorder.h`](../xrRender/Blender_Recorder.h.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: this is a script binding and a compile-time orchestration; nothing here touches the device. It sits in a T1 module only because its callee does.

## Purpose

A *shader* in this engine's vocabulary is not a GPU program. It is a script — one function per rendering role — that the engine calls at load time to *describe* a material's passes. Those scripts ship with the game and are frozen: this file's exported surface is part of the game's data contract, and every name in it must exist with the same spelling and the same argument list or the shipped material set will not load.

That makes this page a specification rather than an algorithm. The other half of the specification is the compilation order at the bottom — which of a material's five elements get built, from which script function, and under what conditions detail texturing is enabled.

This file is a near-duplicate of the Direct3D 11 backend's equivalent. The split is arbitrary in the sense that the *surface* is identical; it is not arbitrary in that each backend's compiler has slightly different capabilities behind the same names, and the original chose duplication over parameterization. A rebuild should have one binding layer over a backend-agnostic compiler.

## State

`Stateless.` — the file registers a vocabulary and orchestrates compilation. The state it manipulates belongs to the compiler it wraps.

## The sampler vocabulary

**Contract** — the object a script receives from `sampler(name)`. Every method returns the same object so calls chain. If the named sampler does not exist in the pass's programs, the object is inert and every call silently does nothing — which is the normal case, since the standard binding set offers dozens of names and a given pass declares few.

```text
texture(name)          bind a texture by logical name, or by the pass's
                       positional texture argument
project(bool)          mark the coordinate as needing a projective divide
clamp() wrap() mirror()            address mode, all three axes at once
f_anisotropic() f_trilinear()      filter presets: min / mip / mag together
f_bilinear() f_linear() f_none()
fmin_none() fmin_point()           individual filter stages, for materials
fmin_linear() fmin_aniso()         that need an asymmetric combination
fmip_none() fmip_point() fmip_linear()
fmag_none() fmag_point() fmag_linear()
comp_less()            turn this sampler into a depth-comparison sampler
```

**Notes** — `comp_less` is the shadow-map sampler request, and it is the one entry here with no Direct3D 9 ancestry: it sets the two engine-invented sampler states described in [`glState.cpp`](glState.cpp.md). Its presence in the *scripted* surface means the shipped scripts can ask for hardware shadow comparison directly.

## The compiler vocabulary

**Contract** — the object a script's element function receives as its first argument.

```text
begin(vertex_program, pixel_program)                 open a pass
begin(vertex_program, geometry_program, pixel_program)
sorting(priority, strict)      where this element sits in the draw order
emissive(bool) distort(bool) wmark(bool)   element-level flags the render
                                           graph routes on
fog(bool)                      whether the pass participates in fog
zb(test, write)                depth test and write
blend(enable, src_factor, dst_factor)
aref(enable, reference)        alpha-test request (absorbed into a macro)
color_write_enable(r, g, b, a) per-channel colour mask
dx10stencil(enable, func, read_mask, write_mask, fail, pass, zfail)
dx10stencil_ref(value)
dx10atoc(bool)                 alpha-to-coverage
dx10zfunc(func)                override the depth predicate
sampler(name)                  -> a sampler object
dx10Options()                  -> an options object
```

**Invariants** — **opening a pass implicitly closes the previous one.** The compiler object tracks whether any pass has been opened yet; every `begin` after the first ends the pass in progress, and the caller ends the last one after the script returns. A script therefore never closes a pass explicitly, and a rebuild must reproduce that or every material will be short one pass.

**Notes** — the `dx10`-prefixed names are a historical artifact: they were introduced for the first Direct3D 10 backend and the shipped scripts use them. Three of them are also exposed under un-prefixed aliases for newer scripts. A rebuild must keep the prefixed spellings — they are in the data — but is free to add its own aliases.

Two of these are accepted and do not mean on this backend what their names suggest. `aref` does not set a pipeline state; the alpha reference reaches the pixel program as a compile-time macro (see [`glState.cpp`](glState.cpp.md)). `dx10atoc` sets an engine-level state that the multi-sample path consumes.

## The enumeration vocabulary

**Contract** — three enumerations the scripts name values from, exported under the names `blend`, `cmp_func` and `stencil_op`. Their *values* are the frozen Direct3D 9 numbering, because that is what the compiler's state stream speaks and what [`glStateUtils`](glStateUtils.cpp.md) translates from.

```text
blend      : zero, one, srccolor, invsrccolor, srcalpha, invsrcalpha,
             destalpha, invdestalpha, destcolor, invdestcolor, srcalphasat
cmp_func   : never, less, equal, lessequal, greater, notequal,
             greaterequal, always
stencil_op : keep, zero, replace, incrsat, decrsat, invert, incr, decr
```

**Invariants** — these names are in the shipped scripts and are frozen. Their numeric values are internal and may be renumbered by a rebuild, *provided* the same rebuild renumbers the compiler's state stream and its translation tables together.

## The options vocabulary

**Contract** — one query, `dx10_msaa_alphatest_atoc()`, answering whether the renderer is configured for alpha-to-coverage alpha testing. Scripts branch on it to emit a different pass. It is the only device capability the scripts can see.

## `load_shader_scripts`

**Contract** — registers the vocabulary above into the script virtual machine, then loads every script in the backend's shader directory. Each file becomes a namespace named after the file; a file with an empty stem loads into the global namespace. Aborts if the directory does not exist.

```text
FUNCTION load_shader_scripts() -> ()
  script_engine.initialize(register_vocabulary)
  folder := list_files("$game_shaders$" / backend_shader_subdirectory, top level only)
  FAIL IF folder is missing
  FOR EACH file IN folder
    IF file's extension is not the script extension THEN CONTINUE
    namespace := file's stem, or the global namespace when the stem is empty
    load file into namespace
```

**Notes** — the backend's shader subdirectory is what separates the OpenGL shader set from the Direct3D one. This backend names its own (see [`README`](README.md)), and that single string is what selects an entire parallel tree of shader sources *and* material scripts.

## `has_shader(name)`

**Contract** — answers whether a material name resolves to a script with at least one usable element. Path separators in the name become underscores first, because a material's logical name is a path and a script namespace is not. Takes the script lock.

## `create_shader(material_name, texture_list)`

**Contract** — compiles a material into its five elements and interns the result. The texture list is the material's positional texture arguments, parsed from a comma-separated string. Returns an existing structurally-equal material if one exists. Takes the script lock across the whole compilation, then the material-table lock.

```text
FUNCTION create_shader(material_name, texture_list) -> Material
  name := material_name with path separators replaced by underscores
  parse texture_list into the compiler's positional texture slots

  # Element 0 -- the highest detail level. A material may offer a distinct
  # high-quality variant; it is only used when this material's base texture
  # actually has a detail texture paired with it in the texture description.
  IF script defines "normal_hq" THEN
      element_index := 0
      detail_enabled := texture_description.detail_pair_for(texture_list[0])
      element[0] := compile(detail_enabled ? "normal_hq" : "normal")
  ELSE IF script defines "normal" THEN
      element_index := 0
      detail_enabled := texture_description.detail_pair_for(texture_list[0])
      element[0] := compile("normal")

  IF script defines "normal" THEN
      element_index := 1; detail_enabled := (same query)
      element[1] := compile("normal")            # the lower detail level

  IF script defines "l_point"   THEN element[2] := compile("l_point",   no detail)
  IF script defines "l_spot"    THEN element[3] := compile("l_spot",    no detail)
  IF script defines "l_special" THEN element[4] := compile("l_special", no detail)

  RETURN intern(material)
```

**Invariants** — the five element slots are a fixed vocabulary the render graph indexes by role: two level-of-detail variants of the base appearance, then the point-light, spot-light and special passes. The numbering is frozen by the render graph, not by the data.

**Invariants** — the detail-texture query runs *per element* and only for the two base elements; the light passes never get detail texturing. The high-quality element falls back to the ordinary one when the base texture has no detail pairing, which means the same script function can serve both and the difference is data-driven.

## `compile_element(namespace, function_name)`

**Contract** — runs one script element function and interns the resulting element. Prepares three positional arguments — the first texture, the second texture (or a null name), and the detail texture (or a null name) — invalidates the state recorder, calls the script, closes the trailing pass, and hands the element to the intern table.

**Notes** — the three positional arguments are the whole data interface between a material's texture list and its script. A script that wants a fourth texture must name it, and the standard binding set then has to know that name. This is why the shipped material scripts are so uniform: they were written against a three-argument convention.
