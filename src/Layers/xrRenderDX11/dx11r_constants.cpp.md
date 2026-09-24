# src/Layers/xrRenderDX11/dx11r_constants.cpp

> Turns a compiled program's description of itself into the engine's constant table: every uniform and every bound resource, named, typed, placed, and sorted for lookup.

**Needs** — [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md) · [`xrRender/r_constants.h`](../xrRender/r_constants.h.md) · [`xrRender/ResourceManager.h`](../xrRender/ResourceManager.h.md) · [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Blender_Recorder_R3.cpp`](Blender_Recorder_R3.cpp.md) · [`r4_shaders.cpp`](../xrRenderPC_R4/r4_shaders.cpp.md)
**Tier floor** — T1: it reads the compiler's reflection of a compiled binary and packs stage and slot into bit fields of one integer.

## Purpose

This is where **by-name binding is resolved, once**. The material system names the quantities it wants to set — `m_WVP`, `fog_color`, `s_base` — and never learns where they live. A compiled program knows where everything lives but not what the engine calls it. This file joins the two: it walks the compiled program's self-description and produces, for each named uniform, a record saying *which stages want it, which block it lives in, at what byte offset, and in what shape*; and for each named resource, *which flat slot it binds to*.

Everything downstream — the per-draw setters, the material's constant handlers, the texture binder — works from those records. No string is compared on the per-draw path.

## State

The constant table is declared in chapter 18; what this file fills in is:

```text
RECORD Constant                 # one per distinct name across the whole program set
  name        : text
  type        : enum            # float | int | bool  (element type)
  destination : int (32-bit)    # bit field: see below
  per_stage_load : map<stage, Load>
  sampler_load   : Load         # for resources rather than uniforms

RECORD Load
  index : int (16-bit)          # byte offset inside its block, or a flat resource slot
  cls   : enum                  # shape: 1x1 | 1x2 | 1x3 | 1x4 | 2x4 | 3x4 | 4x4,
                                # or the resource kind
```

The `destination` bit field is the compact encoding everything else decodes:

```text
low bits   : one flag per stage that uses this constant (vertex, pixel, geometry,
             hull, domain, compute), plus a flag meaning "this is a bound resource"
high bits  : per stage, the index of the constant block this constant lives in,
             each stage occupying its own shifted field
```

Invariant: a constant seen in two stages accumulates both stage flags and keeps a *separate* load record per stage, because the same name may sit at different offsets in each stage's blocks.

## `parse`

**Contract** — given a compiled program's reflection and the stage it was compiled for, populates the table. Creates (or reuses) one constant block object per declared block *per submission context*, records each block under a key that packs stage and block index, parses the uniforms inside each block and then the bound resources, and finally sorts the table by name.

```text
FUNCTION parse(reflection, stage) -> ok
  FOR EACH block_index, block IN reflection.constant_blocks
    dest = stage_flags(stage) OR (block_index SHIFTED INTO stage's block-index field)
    parse_uniforms(block, dest)
    FOR EACH context
      table.blocks[context][pack(stage, block_index)] =
          resources.acquire_constant_block(context, block)   # deduplicated; see Notes
  parse_resources(reflection, stage)
  sort table by name          # ascending, so lookup by name is a binary search
```

**Invariants** — The table must end sorted by name; the lookup used by the material system and by `get_ConstantDirect` assumes it. This is the only reason a sort appears at load time.

**Notes** — Acquiring the block through the resource manager rather than constructing it is what makes shared blocks shared: the manager returns an existing block whenever the new one would be *similar* (same name, layout and member names). One block object per submission context exists because contexts upload independently.

## `parseConstants`

**Contract** — walks one block's declared members and turns each into a constant record, merging into any record of the same name already present from another stage. Rejects anything it cannot represent, loudly.

```text
FUNCTION parse_uniforms(block, destination)
  FOR EACH member IN block.members
    element_type = float | int | bool        # anything else: fatal
    offset = member.byte_offset_in_block     # must fit in 16 bits
    shape =
      scalar                     -> 1x1
      vector of 2 / 3 / 4        -> 1x2 / 1x3 / 1x4
      row-major matrix, 4 cols,
        2 / 3 / 4 rows           -> 2x4 / 3x4 / 4x4
      column-major matrix        -> FAIL WITH "unsupported"
      structure                  -> FAIL WITH "unsupported"
      object (sampler etc.)      -> skipped here; handled as a bound resource
    c = table.find(member.name) OR new constant
    c.destination = c.destination OR destination
    c.load_for(destination) = (offset, shape)
```

**Notes** — A one-component "vector" is a hard error rather than being folded into the scalar case: the compiler emits scalars as scalars, so a one-component vector means the reflection data is not what this code believes it is, and silently accepting it would mis-place every later member.

The rejection of column-major matrices and structures is a real constraint on the shipped shader sources, not a gap: the engine's uploader writes matrices in one convention (see [`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md)) and has no path for nested layouts.

## `parseResources`

**Contract** — walks the program's bound-resource table and records each texture, sampler and writable typed resource as a constant of the corresponding kind, with a *flat* slot index. Anything else is ignored. A resource seen from more than one stage must agree in kind and slot, or it is a fatal inconsistency.

**Invariants** — The slot index is the program's own bind point **plus a per-stage base**, so that the six stages' slot spaces are flattened into one index space. That single space is what the texture and resource binder is written against: a caller sets "the texture at flat slot N" without knowing which stage it belongs to. The bases are fixed constants owned by the texture layer; changing one silently rebinds everything.

Each resource is required to occupy exactly one bind point — arrays of textures are not supported.

**Notes** — Resource lookup takes an optional kind alongside the name. In the compatibility mode used for the oldest shipped shader dialect, a texture and its sampler may legitimately share a name (that dialect had one combined object), so the kind disambiguates them; in the modern dialect they must not collide and the kind is left unspecified so a collision is caught.
