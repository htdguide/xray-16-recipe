# src/Layers/xrRender/Blender.cpp

> The base behaviour every material template shares: its identity record, its two universal knobs (sort priority and strict back-to-front), and the rule that only the active render backend may make one.

**Needs** — [`Blender.h`](Blender.h.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`Blender.h`](Blender.h.md); callers name that, not this file.
**Tier floor** — T2: nothing here touches the device. What stops T3 is the identity record, which is written to and read from a shipped binary file as a byte image.

## Purpose

A **blender** is a parameterized material template. It is not a GPU program and it is not a material: it is a small object that, given a compilation context (which textures this material instance names, which renderer generation is running, whether detail texturing is on), *emits* a concrete list of render passes. The shipped game data names a blender by a class identifier and supplies its parameter values; the renderer instantiates the class, loads the parameters, and asks it to compile.

This file holds the part of that contract that every blender shares, and it is deliberately tiny. Everything interesting is in the ~50 concrete subclasses under `blenders/`, each of which knows how one *kind* of surface is drawn.

## State

```text
RECORD BlenderDescription       # written into the shipped material library, byte-for-byte
  class_id  : int (64-bit)      # eight ASCII characters packed into an integer; see Blender_CLSID.h
  name      : text (<=128)      # the material's own name, lowercased, no dot allowed
  computer  : text (<=32)       # authoring machine name, provenance only
  time      : int (32-bit)      # authoring timestamp, provenance only
  version   : int (16-bit)      # parameter-block layout version, NOT overwritten on load
```

```text
RECORD Blender                  # the shared base of every material template
  description       : BlenderDescription
  priority          : int in [0,3], default 1   # coarse draw order bucket
  strict_back_front : bool                      # force painter's-order sorting within the bucket
  base_texture_name : text, default "$base0"    # which of the instance's textures is "the" texture
  base_transform    : text, default "$null"     # which of the instance's matrices animates it
```

**Invariants**

- The material name is lowercased on assignment and may not contain a dot: material names index into the shipped library by exact string, and the dot is reserved because a *texture* name may carry one. A name that fails either rule is an authoring error, not a runtime condition.
- `version` is the one field that survives a load: the record is read as a byte image over the in-memory one, and then the version this build understands is written back over what the file said. The field records the *parameter block's* layout, which the code owns, not the file.
- When `strict_back_front` is set, `priority` must be 2 or 3. The sort key packs the two together (see the draw-stream chapter); back-to-front ordering is only honoured in the upper half of the priority range, so asking for it at priority 0 or 1 silently would not work. The engine asserts rather than correcting.

## `BlenderDescription.setup(name)`

**Contract** — stamps the identity record: the given name lowercased, the host machine's name, and the current wall-clock time. Called when a material is *created* in the authoring tools, never when one is loaded. Allocates nothing.

**Notes** — The machine name and timestamp are pure provenance — nothing reads them at runtime. They exist because the shipped material library was built by a team and "who last touched this material" was worth a byte cost of 36. A rebuild that only *reads* the shipped library can drop both, as long as it still writes the same number of bytes when it writes one back.

## `Blender` — the template contract

**Contract** — the surface every concrete blender must fill, and the defaults it inherits.

```text
INTERFACE Blender
  comment()             -> text        # required: human description, shown in the authoring tools
  name()                -> text        # defaults to description.name

  can_be_detailed()     -> bool        # default false: may a detail texture be layered on this?
  can_be_lightmapped()  -> bool        # default false: does this surface take a baked lightmap?
  can_use_steep_parallax() -> bool     # default false: may the height channel displace it?

  save(writer)                         # parameter block out, as a tagged property list
  load(reader, version)                # parameter block in
  compile(context)                     # emit the pass list into the context
```

**Invariants** — `save` and `load` must mirror each other exactly, including the order of the property markers, because the format is the shipped library's. A subclass that adds a parameter appends to *both* and bumps its own version; a subclass that reads a version it does not know must still consume the right number of bytes.

**Notes** — The three capability queries are asked *before* compilation, by the compiler, to decide what to put in the context. They are not a type system; they are three yes/no questions about whether this kind of surface can carry three specific extra layers. A rebuild is free to make them data on the class rather than virtual answers.

## `Blender.compile(context)` — the base implementation

**Contract** — writes the two universal knobs into the pass list being built, and nothing else. Every subclass calls this (or sets the same two values) before emitting passes.

```text
FUNCTION compile(context)
  context.set_params(priority, strict_back_front)
```

**Notes** — The source has a vestigial branch here: it tests whether the oldest renderer's lightmap and dynamic-light flags are off and then does *the same thing* in both arms. A variable that once distinguished them was dropped and the branch was not. There is nothing to recover — the two paths are identical and a rebuild writes one line.

## `Blender.create(class_id)` / `Blender.destroy(blender)`

**Contract** — the only way to make or unmake a blender. Both delegate to the *active render backend*, which owns the class-identifier-to-constructor table.

**Notes** — This indirection is the whole reason the material system is portable across backends. The same shipped material library names, say, `"MODEL   "`; the forward renderer maps that identifier to a two-pass fixed-function template and the deferred renderer maps it to a single g-buffer write. The data does not change; the table does. A rebuild must keep this seam — the mapping from class identifier to template is **per backend**, not global — or it will need a separate material library per renderer, which the shipped data does not provide.

The allocation itself also belongs to the backend because a blender is allocated out of the renderer's own arena and must die before the renderer does.
