# src/Layers/xrRenderDX11/dx11ResourceManager_Scripting.cpp

> The material description language: the script surface a shipped material file calls to describe its passes, and the five level-of-detail entry points every material is asked for.

**Needs** — [`xrRender/blender_recorder.h`](../xrRender/blender_recorder.h.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [`xrRender/blender.h`](../xrRender/blender.h.md) · [`xrScriptEngine/script_engine.hpp`](../../xrScriptEngine/script_engine.hpp.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it is a declarative binding table plus a load-time driver; nothing here has a frame budget. It sits in a T1 module only because its callees do.

## Purpose

A **shader** in this engine's vocabulary is a *material pass description*, and it ships with the game as a script file. This file defines the vocabulary those scripts may use on this backend, and drives the compilation of one material.

Two things make it load-bearing rather than glue. First, **the script surface is part of the frozen data contract**: every method name here is called by shipped material files, so a rebuild must expose the same names with the same semantics or the game's materials will not load. Second, it encodes **what a material is asked to produce** — five separate compilations per material, one per usage — which is a structural fact about how the renderer draws.

## State

`Stateless.` The binding objects are transient wrappers around the material compiler for the duration of one script call.

## the script surface

The vocabulary divides into three objects. Each method returns its receiver so a material reads as a chain.

**The compiler object** — one pass at a time:

```text
begin(vertex_program, pixel_program)                  # opens a pass; closes the previous one
begin(vertex_program, geometry_program, pixel_program)
sorting(priority, strict)      # where this pass lands in the draw order
emissive(flag) / distort(flag) / wmark(flag)   # pass classification the renderer branches on
fog(flag)
zb(test, write)                # depth test and depth write
blend(enable, source_factor, destination_factor)
aref(enable, value)            # alpha cut-off (emulated as a constant; see the state manager)
color_write_enable(r, g, b, a)
dx10texture(resource_name, texture_name)   # bind a texture to a named shader resource
dx10stencil(enable, func, read_mask, write_mask, fail_op, pass_op, depth_fail_op)
dx10stencil_ref(value)
dx10cullmode(mode)
dx10zfunc(func)
dx10atoc(enable)               # alpha-to-coverage, meaningful only with multisampling
sampler(name) -> sampler object
dx10sampler(name) -> sampler object      # declares a sampler with no texture of its own
dx10Options() -> options object
```

**The sampler object** — addressing and filtering for one sampler slot: `texture`, `project`, `clamp`/`wrap`/`mirror`, the composite filters (`f_anisotropic`, `f_trilinear`, `f_bilinear`, `f_linear`, `f_none`) and the per-stage filter setters (`fmin_*`, `fmip_*`, `fmag_*`).

**The options object** — what a material may ask about the current configuration: whether alpha testing is being resolved through alpha-to-coverage, and the name of the level being loaded. That second one exists so a material can special-case a specific map, which is a content-side escape hatch and a real dependency: a rebuild that does not expose the level name will silently change how some shipped materials render.

Three enumeration tables are exported alongside — blend factors, comparison functions, stencil operations — so material files name them symbolically.

**Notes** — The `dx10`-prefixed names are historical: they were the newer backend's additions when the older one still existed. Several are also exported under their unprefixed names for compatibility. A rebuild must keep both spellings, because shipped material files use both.

The older combined `sampler(name)` declares a sampler *and* the texture bound with it; the newer pair `dx10sampler(name)` plus `dx10texture(resource, texture)` separates them, matching a shader model where samplers and textures are distinct objects. Both are live.

## `LS_Load`

**Contract** — registers the binding surface with the script virtual machine, then loads every material script in the backend's shader directory, each into a namespace named after its file. Called once at renderer start-up.

**Notes** — The shader directory is *per renderer*: each backend asks its own implementation for the path, and the game data ships a directory per renderer generation. This is the hook a rebuild uses to ship translated materials for a different graphics API without touching the loader.

## `_lua_Create` — compiling one material

**Contract** — given a material name and its texture list, produces the material's five compiled elements. Interns the result: an identical material already built is returned instead. Takes the script lock for the compilation and the shader-registry lock for the interning, because materials are compiled from more than one thread during level load.

```text
FUNCTION compile_material(name, textures) -> Material
  name = name with path separators replaced by underscores   # scripts are flat namespaces
  elements[0] high-detail  : call "normal_hq" if it exists, else "normal"
                             with the detail-texture pairing resolved for texture 0
  elements[1] low-detail   : call "normal", same pairing
  elements[2] point light  : call "l_point"   (no detail pairing)
  elements[3] spot light   : call "l_spot"    (no detail pairing)
  elements[4] special      : call "l_special" (no detail pairing)
  RETURN intern(elements)
```

**Invariants** — The five slots are fixed and the renderer indexes them directly: the first two are the material drawn at its two levels of detail, and the last three are how the same surface responds when it is lit by a point light, by a spot light, or by the special pass (shadow and depth-only work). A material that does not define an entry leaves that slot empty, and the renderer skips it. This five-way split *is* the renderer's lighting architecture expressed in the data format, and it is frozen.

**Invariants** — The **detail-texture pairing** is resolved before the high- and low-detail elements are compiled, and only for them: the engine looks up whether the material's first texture has an associated detail texture and scale, and if so compiles a variant that layers it. Lighting elements never get detail. The pairing table is part of the frozen texture data.

## `_lua_HasShader`

**Contract** — answers whether a material script defines either the ordinary or the special entry point, without compiling. Used to decide whether a surface can be drawn at all.

## `_lua_Compile`

**Contract** — invokes one entry point of one material script with the compiler wrapper and up to three texture names (the base, the second, and the detail texture, each "null" when absent), closes the final pass, and interns the resulting element.

**Notes** — Passes are closed *lazily*: opening a pass closes the previous one, and the driver closes the last one after the script returns. That is why a material script never says "end pass", and a rebuild's compiler must do the same or every shipped material will be one pass short.
