# src/Layers/xrRender/ResourceManager_Scripting.cpp

> The material definitions that ship as scripts: the declarative surface they are written against, how a script file becomes a namespace, and how one named function in it becomes a compiled pass list.

**Needs** — [`ResourceManager.h`](ResourceManager.h.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`Blender.h`](Blender.h.md) · [`Shader.h`](Shader.h.md) · [`tss.h`](tss.h.md) · [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`xrScriptEngine/script_engine.hpp`](../../xrScriptEngine/script_engine.hpp.md) · [`xrScriptEngine/script_space.hpp`](../../xrScriptEngine/script_space.hpp.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md)
**Tier floor** — T2: it is name resolution and a builder pattern over the pass recorder. Nothing here touches bytes or the device directly.

## Purpose

The modern renderers do not describe their materials in compiled C++ templates. They describe them in **script files that ship with the game data**, one file per material family, each defining a handful of functions that emit passes. This file is that mechanism: the declarative surface exported into the interpreter, the rule that turns a file into a namespace, and the compilation that calls one function per render purpose.

Everything named on this page is **frozen**. The shipped script files call these methods by these names; renaming any of them breaks materials the engine did not write.

## State

`Stateless` in itself. It owns the registry's script interpreter and the lock around it — a *separate* interpreter from the game's Lua state, sharing only the language seam. Material compilation runs from several threads, so every call into the interpreter is serialized.

## The shipped material script convention

```text
LOCATION   $game_shaders$/<renderer's shader directory>/*.s
           # root of that directory only; subdirectories are not scanned

NAMESPACE  the file's name without its extension.
           A file whose name is empty maps to the global namespace.

A MATERIAL is a namespace containing up to five functions, each named for
a render purpose.  All are optional; a namespace with none is not a material.

  normal_hq   the detailed variant   -> compiled into slot 0, only when the base
                                        texture has a detail pairing
  normal      the plain variant      -> slot 0 when there is no detail pairing,
                                        and always slot 1
  l_point     point-light purpose    -> slot 2
  l_spot      spot-light purpose     -> slot 3
  l_special   the fifth purpose      -> slot 4
  editor      authoring-tool variant -> used instead of the above in the tools

EACH FUNCTION receives four arguments, in this order:
  compiler       the pass-building object described below
  base texture   the material's first texture name, or "null"
  second texture the material's second texture name, or "null"
  detail texture the paired detail texture name, or "null"
```

**Invariants**

- A material *exists* — in the sense that `Create` will route to the script path — if and only if its namespace contains `normal` or `l_special` (in the authoring tools: `editor`). Those two are the existence test; the others are optional refinements.
- A material name containing the platform's path separator is **undecorated** before it is looked up: every separator becomes an underscore. Shipped level data names materials with paths (a material under a directory), and a namespace cannot contain a separator. This transform is frozen against the shipped data and must be reproduced exactly, including that it is applied to the terminating position as well, so a name is never left unterminated.
- Slot 5 is never produced by a script. Only the compiled-template path fills it.
- Unfilled slots stay empty, and every draw path must tolerate an empty slot for a purpose it does not use.

## `LS_Load`

**Contract** — brings up the material interpreter, exports the declarative surface into it, and loads every material script in the renderer's shader directory, each into its own namespace. Runs once, before the binary material library is read. Blocks.

```text
FUNCTION load_material_scripts()
  start the interpreter, exporting the sampler, compiler and blend-mode surfaces
  FOR EACH file IN $game_shaders$/<renderer's shader directory>, root only
    IF the file's extension is not ".s"
      CONTINUE
    namespace = the file name without its extension, or the global namespace if empty
    load the file into that namespace
```

**Notes** — Each file gets its own namespace rather than one shared global scope, so two material families may both define `normal` without colliding. That is what makes the file name the material's *identity* and the function name the material's *purpose* — a two-level naming scheme carried entirely by the filesystem and the language's scoping.

## The declarative surface — `_compiler`

**Contract** — the object every material function receives first. Each method returns the object again so calls chain; the chain reads as a declaration of one pass. Each method forwards to the pass recorder (see [`Blender_Recorder.h`](Blender_Recorder.h.md)).

```text
OBJECT compiler
  begin(vertex_program, pixel_program)  start a pass, naming its two programs.
                                        Fog is enabled for the pass.
  sorting(priority, strict_back_to_front)
                                        the element's coarse draw bucket (0..3) and
                                        whether painter's order is forced within it
  emissive(bool)                        this element is self-lit
  distort(bool)                         this element writes into the screen-distortion target
  wmark(bool)                           this element is a wallmark (a decal)
  fog(bool)                             per-pass fog override
  zb(test, write)                       depth test and depth write, independently
  blend(enabled, source_factor, destination_factor)
  aref(enabled, threshold)              alpha-reference test and its cutoff
  color_write_enable(r, g, b, a)        per-channel colour mask
  sampler(name) -> sampler              bind a sampler by its name in the program,
                                        yielding the sampler object below
```

**Invariants** — `sorting` is the only method whose effect is on the *element* rather than the pass: it sets the priority and back-to-front flags that become part of the draw-order key, and those are properties of the whole element. So are `emissive`, `distort` and `wmark`. A material that calls them inside its second pass still sets them for the element; the surface does not prevent it.

## The declarative surface — `_sampler`

**Contract** — returned by `sampler(name)`. Also chains. A sampler whose name is not present in the pass's programs yields an **inert object**: every method on it does nothing. That is how one material function serves several programs that bind different subsets of the textures — the script declares every sampler it might need, and the ones the program does not have are silently dropped.

```text
OBJECT sampler
  texture(name)      bind a texture to this sampler
  project(bool)      the coordinates are projective and must be divided through

  clamp() wrap() mirror()             addressing mode, all axes

  f_anisotropic()    min anisotropic, mip linear, mag anisotropic
  f_trilinear()      min linear,      mip linear, mag linear
  f_bilinear()       min linear,      mip point,  mag linear
  f_linear()         min linear,      mip none,   mag linear
  f_none()           min point,       mip none,   mag point

  fmin_none() fmin_point() fmin_linear() fmin_aniso()   minification, individually
  fmip_none() fmip_point() fmip_linear()                mip selection, individually
  fmag_none() fmag_point() fmag_linear()                magnification, individually
```

**Invariants** — the five combined filter names are the ones shipped materials use; the nine individual setters exist for the cases that do not fit. The combinations are frozen: `f_bilinear` means point mip selection, not linear, and a rebuild that "corrects" it changes the appearance of every material that asks for it.

**Notes** — The inert-object trick is the load-bearing idea here. The alternative — asking the script to test whether a sampler exists — would put device knowledge into shipped data. Making the missing case a silent no-op keeps the script declarative. A rebuild must reproduce the *silence*: a script binding a sampler the program lacks is normal, not an error.

## The declarative surface — `blend`

**Contract** — an enumeration of blend factors, exported under the name `blend`, for the arguments of `blend(...)`. The names are frozen:

```text
zero  one  srccolor  invsrccolor  srcalpha  invsrcalpha
destalpha  invdestalpha  destcolor  invdestcolor  srcalphasat
```

**Notes** — These are the classic fixed-function blend factors of the graphics API of the era, exposed under short names. Their *values* are that API's numbering and travel through to the state recorder; a rebuild on another API must map the names, not the numbers. `srcalphasat` — source alpha saturated — is the one that has no direct equivalent everywhere and must be checked.

## `_lua_HasShader`

**Contract** — tests whether a material name resolves to a scripted definition. Undecorates the name, then asks the interpreter whether the resulting namespace holds a function named `normal` or `l_special` (in the authoring tools, `editor`). Takes the interpreter lock. Cheap; called for every material before compilation.

## `_lua_Create`

**Contract** — compiles a scripted material into a shader. Undecorates the name, parses the texture list, then compiles up to five elements, each by calling one named function. Takes the interpreter lock for the whole compilation and the shader-registry lock for the interning. Returns an interned shader.

```text
FUNCTION compile_from_script(material_name, textures) -> Shader
  namespace = undecorate(material_name)         # path separators -> underscores
  context.textures = parse_list(textures)
  context.template = none                       # no compiled template is involved
  context.fixed_function = false                # scripted materials are programmable-pipeline only

  LOCK the interpreter DURING
    # slot 0 — the detailed variant, if the base texture has a detail pairing
    IF namespace has "normal_hq"
      context.purpose = 0
      context.detail  = detail_pairing_for(context.textures[0])
      slot[0] = compile(namespace, context.detail ? "normal_hq" : "normal")
    ELSE IF namespace has "normal"
      context.purpose = 0
      context.detail  = detail_pairing_for(context.textures[0])
      slot[0] = compile(namespace, "normal")

    # slot 1 — the plain variant, always from "normal"
    IF namespace has "normal"
      context.purpose = 1
      context.detail  = detail_pairing_for(context.textures[0])
      slot[1] = compile(namespace, "normal")

    # slots 2, 3, 4 — the light and special purposes, never detailed
    FOR EACH (slot_index, function_name) IN ((2, "l_point"), (3, "l_spot"), (4, "l_special"))
      IF namespace has function_name
        context.purpose = slot_index
        context.detail  = false
        slot[slot_index] = compile(namespace, function_name)

  LOCK the shader registry DURING
    RETURN intern_shader(slot)
```

**Invariants**

- Slot 0 falls back to `normal` when `normal_hq` exists but the base texture has no detail pairing. The two are not "high and low quality" in the abstract — `normal_hq` specifically means *the variant that layers a detail texture*, and compiling it without one would bind a sampler to nothing.
- Slots 2 through 4 never receive a detail texture, because their purposes — light accumulation and shadow generation — do not sample the surface's colour.
- The interpreter lock spans all five compilations, not each one. A material must compile against a consistent interpreter state; interleaving two materials' compilations is not safe because the compilation context is reached through a global.

**Notes** — The two locks are taken in sequence, never nested, and in a fixed order. That the shader registry needs its own lock at all is a consequence of materials compiling on several threads; the by-value interning scan and the append must be atomic together, or two threads produce two identical shaders.

## `_lua_Compile`

**Contract** — compiles one element by calling one script function. Prepares an element to record into, invalidates the recorder's state, resolves the three texture arguments, calls the function with a compiler object and those three names, closes the recording, and interns the element.

```text
FUNCTION compile(namespace, function_name) -> optional<ShaderElement>
  element = a fresh, empty element
  point the recorder at it; invalidate its accumulated state

  base_texture   = textures[0] if non-empty else "null"
  second_texture = textures[1] if the list is longer than one else "null"
  detail_texture = the paired detail texture if any else "null"

  call namespace.function_name(compiler_for(recorder),
                               base_texture, second_texture, detail_texture)
  close the recording                        # flushes the last pass
  RETURN intern_element(element)
```

**Invariants** — `"null"` is the frozen spelling for an absent texture argument, and scripts test for it. It is the same spelling the texture creator treats as "no texture", so a script may pass it straight through to `texture(...)` without checking.

**Notes**

- Exactly three texture arguments are passed, no matter how many the material named. A material needing a fourth texture names it inside the script, not through the argument list. The three chosen are the ones whose *identity varies per material instance*: the base, its companion, and the detail pairing that the engine — not the script — decided on.
- The recorder's state is invalidated before the call rather than after. State accumulated by a previous element must not leak into this one, and invalidating at the start rather than the end means a compilation abandoned midway cannot poison the next.
