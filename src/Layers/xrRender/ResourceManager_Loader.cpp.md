# src/Layers/xrRender/ResourceManager_Loader.cpp

> Reads the shipped material library — a chunked file of animator definitions and blender parameter blocks — into the registry, and tears the same set down when the device goes away.

**Needs** — [`ResourceManager.h`](ResourceManager.h.md) · [`Blender.h`](Blender.h.md) · [`Blender_CLSID.h`](Blender_CLSID.h.md) · [`SH_Matrix.h`](SH_Matrix.h.md) · [`SH_Constant.h`](SH_Constant.h.md) · [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the blender parameter blocks are read as byte images over in-memory records, and their layout is versioned by the record itself.

## Purpose

Every material the game uses is described in one shipped file, conventionally named for the engine that wrote it. This file is the reader for that format, and the matching teardown. It is separate from the rest of the registry because the *format* is a frozen external contract and the registry's other files are not.

The format is a small container of three chunks, and only the third is interesting.

## State

`Stateless`; it fills the registry described in [`ResourceManager.h`](ResourceManager.h.md).

## The material library format

```text
FILE material library                # "shaders.xr" in the shipped data
  first 8 bytes                      # read before anything else, see below
  CHUNK 0 : constant animator definitions   # a flat sequence, until the chunk ends
      REPEAT
        name    : text, zero-terminated
        payload : the animator's own parameter block
  CHUNK 1 : matrix animator definitions     # identical shape to chunk 0
      REPEAT
        name    : text, zero-terminated
        payload : the animator's own parameter block
  CHUNK 2 : blenders                        # a container of sub-chunks, ids 0,1,2,... contiguous
      SUB-CHUNK n :
        description : BlenderDescription    # fixed-size byte image; see Blender.cpp
        payload     : the blender's parameter block, layout selected by description.version
```

**Invariants**

- The blender sub-chunks are numbered from zero with no gaps: reading stops at the first id that is absent, so a gap silently truncates the library.
- Each blender sub-chunk is read **twice**: once as a fixed-size description to learn the class identifier and the version, then rewound to offset zero and handed whole to the instantiated class. The class re-reads the description itself. A rebuild that parses fields rather than mapping structs simply parses the description once and passes it along.
- Blender names are unique across the library. The engine asserts on a duplicate rather than taking either one — a duplicate means two materials would silently share one definition, which is a data error the authoring tools should have caught.
- The description's version is the *parameter block's* layout version, and the class decides how to read the payload from it. A class whose own version differs from the file's is loaded anyway, with a warning: the class is expected to handle older layouts, and refusing would make the engine less compatible than the one it replaces. Only the non-shipping build logs the mismatch.

## `OnDeviceCreate`

**Contract** — two overloads: one takes an open reader, one a path. The path overload checks the first eight bytes against a known marker and fails hard on a match. Both return immediately if the device is not ready. Blocks. Called once per device creation, before any material is compiled.

```text
FUNCTION load_library(path)
  reader = open(path)
  IF the first 8 bytes are the compressed-library marker
    FAIL WITH "unsupported blender library"
  load_library(reader)

FUNCTION load_library(reader)
  IF the device is not ready
    RETURN
  load the scripted material definitions       # see ResourceManager_Scripting.cpp
  FOR EACH (name, payload) IN chunk 0
    create_constant_animator(name).load(payload)
  FOR EACH (name, payload) IN chunk 1
    create_matrix_animator(name).load(payload)
  FOR EACH sub-chunk IN chunk 2
    description = read the fixed-size description
    blender = instantiate the class named by description.class_id
    IF blender is none
      report "renderer doesn't support this blender"; CONTINUE
    rewind; blender.load(payload, description.version)
    ASSERT the name is not already registered
    register blender under description.name
  load the texture description data            # detail and bump pairings
```

**Notes**

- **The marker check is a refusal, not a capability.** A later variant of the tool chain could write the library compressed, stamped with an eight-byte identifier. This engine does not read it, and says so loudly rather than misparsing. A rebuild only needs the check if it wants the same honest failure.
- **An unsupported blender class is skipped, not fatal.** The library is shared across renderer generations, and each generation registers only the classes it can draw. Materials naming a skipped class fall back at compile time (see [`ResourceManager.cpp`](ResourceManager.cpp.md)). This is the mechanism that lets one shipped data set feed several renderers.
- The names are duplicated into registry-owned storage when registered, because the reader's buffer goes away when the chunk closes. What survives that detail is the ownership rule: the registry's keys outlive the file.
- The scripted definitions are loaded *before* the binary library, so that a scripted material can shadow a compiled one of the same name. The dispatch in `Create` prefers the script, and loading in this order means the script is available by the time the first material compiles.

## `OnDeviceDestroy`

**Contract** — releases the animators, the blenders, the detail-texture table and the scripted definitions. Returns immediately unless the device is already down — the condition is inverted from what the name suggests, and this is checked, not assumed. Asserts that each animator has exactly one reference left, which is the registry's own.

```text
FUNCTION unload()
  IF the device is still ready
    RETURN                                   # called defensively; only act after the device is gone
  unload the texture description data
  FOR EACH animator IN matrices, constants
    ASSERT its reference count is 1          # only the registry's own reference remains
    destroy it
  clear both maps
  destroy every blender and its key
  destroy the detail-texture table's entries
  unload the scripted definitions
```

**Invariants** — the assertion on the reference count is the real content here. The animators are the one resource kind the registry itself holds a reference to; if any material still references one at teardown, the shader registry was not drained first, and the teardown order is wrong. A rebuild gets the same guarantee free if teardown is ordered, but should keep the check: the failure is otherwise silent and shows up much later.

**Notes** — Textures are not released here. They outlive the material library because the texture set survives a device reset, and because the "necessary" set below is deliberately pinned across level transitions.

## `StoreNecessaryTextures` · `DestroyNecessaryTextures`

**Contract** — pin a subset of the currently registered textures by taking a reference to each, so that they survive a level unload; and release the pins. Idempotent: storing when something is already stored does nothing.

```text
FUNCTION store_necessary()
  IF something is already pinned
    RETURN
  FOR EACH texture name IN the registry
    IF the name contains the path segment "levels"
      CONTINUE                       # level-specific: it goes away with the level
    IF the name contains no path separator at all
      CONTINUE                       # a bare name: generated, or a render target, not a file
    pin it
```

**Notes**

- The two exclusions define what "necessary" means by elimination: a texture is worth pinning if it comes from a directory *other* than the level's. Those are the shared textures — user interface, weapons, effects — that the next level will want immediately, and reloading them is the bulk of a level transition's cost.
- The path-segment test is on the *virtual* path as it was named, so the convention is frozen against shipped data: level textures are addressed under a `levels` directory, and nothing else is.
- A bare name with no separator is either a synthesized texture or a render target. Neither has a file to reload, so pinning would be meaningless.
