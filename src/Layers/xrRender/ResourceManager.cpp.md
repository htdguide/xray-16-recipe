# src/Layers/xrRender/ResourceManager.cpp

> Turns a material name plus a list of texture names into a shader — six compiled element variants, interned so that two objects wearing the same material share one.

**Needs** — [`ResourceManager.h`](ResourceManager.h.md) · [`Shader.h`](Shader.h.md) · [`Blender.h`](Blender.h.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`tss.h`](tss.h.md) · [`Texture.cpp`](Texture.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: name resolution, interning and compilation orchestration. The T1 pressure is one level down, in the pass recorder and the texture loader.

## Purpose

The rest of the engine knows materials only as strings: a mesh carries the name of its material and the names of its textures, and that is all the shipped model files record. This file closes that gap. It resolves the material name to a template, hands the template the texture names and a compilation context, collects the pass lists it emits for each *render purpose*, and interns the result.

It is a separate file from the other four only by topic. The one genuinely separable idea in it is [`_ParseList`](#_parselist), which defines the frozen syntax of a material's texture list.

## State

`Stateless` in itself; it reads and writes the registry described in [`ResourceManager.h`](ResourceManager.h.md).

## `Create`

**Contract** — the engine's entry point for every material. Takes a material name and three comma-separated name lists — textures, constants, matrices — and returns an interned shader, or nothing. Blocks: a cache miss compiles, which may load textures and compile programs. Returns nothing unconditionally when the process is a dedicated server, because a dedicated server has no device and no materials. Two overloads exist; the second takes an already-resolved template, which the engine uses for materials it builds itself rather than loads.

```text
FUNCTION create(material_name, textures, constants, matrices) -> optional<Shader>
  IF running as a dedicated server
    RETURN none                        # no device, no materials, and callers expect none

  IF a scripted definition exists under this name
    RETURN compile_from_script(material_name, textures)

  result = compile_from_template(material_name, textures, constants, matrices)
  IF result exists
    RETURN result

  # Nothing matched. Rather than fail a level load for one missing material,
  # substitute a named stand-in that every shipped data set contains.
  IF a scripted definition named "stub_default" exists
    RETURN compile_from_script("stub_default", textures)
  FAIL WITH fatal                      # the stand-in itself is missing: the data set is broken
```

**Notes**

- **Scripted first, template second.** The scripted definitions are the newer mechanism and are how the modern renderers describe their materials; the compiled templates are the older one and remain because the oldest renderer generation and the authoring tools still use them. On the oldest generation the priority inverts: a template is preferred whenever the fixed-function path is active and a template exists, because on that path the template can emit fixed-function state a script cannot express.
- **`stub_default` is a frozen name.** It is the material every unresolvable name falls back to, and it must exist in the shipped shader scripts. The fallback exists because material names in shipped level data outlive the materials themselves — levels reference materials that later data sets dropped — and refusing to load such a level would make the engine less compatible than the one it replaces.

## `_cpp_Create`

**Contract** — compiles a material through a blender template. Resolves the template by name (reporting a miss and returning nothing), parses the three name lists, then compiles the template once per render purpose and interns the resulting element list as a shader. Allocates; blocks on texture loading.

```text
FUNCTION compile_from_template(name, textures, constants, matrices) -> optional<Shader>
  template = blenders[name or "null"]
  IF template is none
    report "not found in library"; RETURN none

  context.template   = template
  context.fixed_function = whether the device is running the fixed-function path
  context.textures   = parse_list(textures)
  context.constants  = parse_list(constants)
  context.matrices   = parse_list(matrices)

  FOR EACH purpose IN the six render purposes
    context.purpose = purpose
    context.detail  = detail_pairing_for(context.textures[0])   # see below
    element = template.compile(context)
    shader.elements[purpose] = intern_element(element)

  RETURN intern_shader(shader)
```

### The six render purposes

A shader is not one pass list, it is six — the same surface drawn for six different reasons. The slot numbering is frozen: it indexes an array in every draw-submission path in the renderer, and the two renderer generations reuse the same six slots for different meanings.

```text
slot   oldest generation                         modern generations
  0    normal, high quality (detail layered on)  deferred g-buffer fill
  1    normal, low quality                       normal, low quality
  2    additive pass for one point light         point-light shadow map
  3    additive pass for one spot light          spot-light shadow map
  4    lighting/shadow contribution of models    directional shadow map
  5    (unused by name; compiled without detail) distortion / self-illumination
```

**Notes**

- **Slot 4 is compiled with detail texturing forced on**, regardless of whether the base texture actually has a detail pairing. The source marks this as a deliberate hack. The reason is visible in the table: on the oldest generation slot 4 is the pass that writes models' lighting contribution, and that pass shares a program with the detailed variant; compiling it without detail would produce a program that does not match the vertex data the pass is fed. A rebuild that separates the two concerns does not need the override.
- **Slot 5 is compiled with detail off** and has no name in the enumeration. It is reserved, and the comments in the type's declaration propose uses for it — night vision, lightmap capture — that were never taken. It is still compiled and still interned, so a rebuild must produce it or the shader's value identity changes.
- The detail pairing is re-queried before every slot even though it depends only on the first texture and cannot change between slots. Re-querying is free and the repetition is incidental.

### Detail pairing

The first name in a material's texture list is *the* texture — the base. The shipped texture description data may pair that base with a **detail texture** and a scaling factor, and when it does, the template is told so and emits a second texture stage layering the detail. This pairing is a named convention in shipped data, not a property of the material: the same material is detailed on one surface and not on another purely because of which base texture it was given. See [`TextureDescrManager.h`](TextureDescrManager.h.md) for the pairing table's own format.

## `_ParseList`

**Contract** — splits a comma-separated list of resource names into normalized names. Pure; writes into a caller-supplied list. An empty or absent input yields the single name `$null`.

```text
FUNCTION parse_list(names) -> list<text>
  IF names is absent or empty
    names = "$null"                   # the frozen spelling of "nothing here"
  FOR EACH field IN split(names, ",")
    field = lowercase(field)
    strip a trailing ".tga", ".dds", ".bmp" or ".ogm"
    emit field
```

**Invariants** — the list is never empty: downstream code indexes element zero unconditionally to find the base texture, and `$null` is what makes that safe.

**Notes**

- The extension stripping accepts four spellings, two of which are for formats the game has not shipped in since before release. The shipped material data still names textures with those extensions, so all four must be accepted. The same normalization is applied at texture *load* time, which is why the two are kept in step.
- Names are lowercased but paths are not otherwise normalized. The filesystem layer does case-insensitive matching (chapter 6), so lowercasing here is about the *interning key*, not about finding the file: two materials naming the same texture with different capitalization must land on one texture object.
- There is no escaping and no quoting. A resource name may not contain a comma. This is frozen by the shipped data, which relies on it.

## `_CreateElement`

**Contract** — interns a pass list. An element with no passes interns to nothing. Otherwise a linear scan finds an equal existing element and returns it, or the prototype is moved into the registry and flagged as registered.

**Invariants** — equality is by value over the pass list *and* the sort flags (priority, strict back-to-front, emissive, distortion, wallmark). Two elements with the same passes but different sort flags are different elements, because the flags become part of the draw-order key and the renderer's bucketing reads them from the element.

## `_cpp_Create` (template overload) · `Delete`

**Contract** — the overload taking an already-resolved template skips the name lookup; everything else is identical. `Delete` removes a shader from the registry, ignoring one that was never registered, and logs when a registered shader cannot be found.

## `CompatibilityCheck`

**Contract** — reads one shipped shader source file and sets a renderer-wide flag according to what it finds. Runs once, at device creation, and is fatal if the file is absent.

```text
FUNCTION compatibility_check()
  source = read the shipped shader source "skin.h"
  region = the text between "u_position" and "sbones_array"
  IF that region is absent
    region = the text between "skinning_pos" and "skinning_0"
  IF region contains "12." then "/" then "32768."
    high_quality_skinning = false    # the data set uses the low-precision bone weights
  ELSE
    high_quality_skinning = true
```

**Notes** — This is the recipe's clearest example of a decision that a rebuild must reproduce but should be ashamed of. A widely used community patch changes the skinned-vertex format to carry more precision, and changes this shader source accordingly; the engine has no version field to ask, so it *reads the shader source as text* and looks for the divisor of the old quantization. Finding `12 / 32768` means the old format. The flag it sets changes how skinned vertices are packed for the device (see [`SkeletonX.cpp`](SkeletonX.cpp.md)), so getting it wrong produces geometry that is subtly wrong everywhere.

A rebuild needs the same *outcome* — detect which of two vertex quantizations the shipped shaders expect — and should reach it the same way only because there is no other signal available. If a rebuild controls the shader sources, it should stamp a version instead.

## `DeferredUpload` · `DeferredUnload`

**Contract** — upload (or release) the pixel data of every registered texture. Both return immediately if the device is not ready. Called around level transitions, not per frame.

**Notes** — Texture *creation* registers a name and reads only the header; the decode and upload are deferred until one of these runs. The point is batching: a level's several thousand textures are uploaded in one pass, which on the backend that permits it is run across worker threads. The backend that does not permit it runs the same loop serially — not because serial is correct, but because that API binds its device context to one thread and the work has never been re-plumbed. A rebuild should treat parallel upload as the intent.

## `_GetMemoryUsage` · `_DumpMemoryUsage`

**Contract** — total the registered textures' memory, split into lightmaps and everything else; and dump a per-texture report sorted by size, with reference counts.

**Notes** — Lightmaps are identified by the substring `lmap` in the texture's name. That is a naming convention in shipped level data, relied on here for accounting only. The split exists because lightmap memory scales with the level and base-texture memory scales with the data set, and a level that will not fit needs to know which of the two grew.

## `open_shader`

**Contract** — opens a file from the shader source directory of whichever renderer is running. The directory name is supplied by the backend, under a single logical root, so that the two backends' incompatible shader dialects can live side by side in the same shipped data.

## `Evict`

**Contract** — a no-op. It was the hook for releasing device memory under pressure on an API that could report pressure. Kept because the frame loop calls it; a rebuild deletes it or fills it.
