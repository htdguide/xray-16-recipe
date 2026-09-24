# src/Layers/xrRenderGL/glr_constants.cpp

> Builds a pass's constant table by asking a linked program what it declares — the by-name binding model, and the one place sampler slots are assigned.

**Needs** — [`glr_constants_cache.h`](glr_constants_cache.h.md) · [`xrRender/r_constants.h`](../xrRender/r_constants.h.md) · [`xrRender/SH_Texture.h`](../xrRender/SH_Texture.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glResourceManager_Resources.cpp`](glResourceManager_Resources.cpp.md) · [`glr_constants_cache.h`](glr_constants_cache.h.md)
**Tier floor** — T1: it reads a driver's reflection data and caches device uniform locations.

## Purpose

The engine binds shader constants **by name**. A material pass says "set `m_shadow`" and something has to know where `m_shadow` lives. On the Direct3D 11 path that mapping comes from a compiler-produced reflection blob and resolves to a constant-buffer slot and byte offset. On this path it comes from the driver: after a program links, it can be asked to enumerate its active uniforms, and each answer carries a name, a type, an array size and a location.

This file turns that enumeration into the engine's constant table. It is the *reflection* half of the constant model; the *writing* half is [`glr_constants_cache.h`](glr_constants_cache.h.md).

Two decisions here shape everything downstream. First, a constant is identified by name across every stage, so a name that appears in both the vertex and the pixel program becomes one table entry with two recorded locations, and setting it writes both. Second, **there are no constant buffers on this backend** — each constant is written individually to its program. The original marks that as work not done, and it is the largest single performance difference between the two backends.

## State

```text
RECORD Constant                  # one entry of a pass's table
  name        : text
  type        : { float, int, bool, sampler }
  destination : bit set over { pixel, vertex, geometry, whole-program }
  binder      : optional<producer>    # only samplers have one here
  loads       : one Load per destination bit

RECORD Load
  index    : int    # the enumeration index; for samplers, also the texture slot
  class    : { scalar, 2-vector, 3-vector, 4-vector, 2x4, 3x4, 4x4 }
  location : int    # the device's own handle for this uniform
  program  : int    # WHICH program object that location belongs to

# Invariant: a location is only meaningful together with its program. With
# separable programs three stages have three independent location spaces, so
# storing the location alone is not enough -- this is why Load carries the
# program handle, and it is the difference from the other backend's model.
# Invariant: the table is kept sorted by name, because lookup is a binary
# search on the name.
```

## `parse(program, destination)`

**Contract** — enumerates a linked program's active uniforms and merges them into the table. The destination says which stage's slot each entry's location belongs to — or, for a monolithic linked program, the whole-program slot. Merges rather than replaces: a name already present gains another destination and another recorded location. Sorts the table by name on the way out. Allocates; runs once per program, at material-compile time. Faults on a uniform type it does not model.

```text
FUNCTION parse(program, destination) -> bool
  FOR EACH active uniform i IN program
    read name, element count, declared type

    # An array uniform is enumerated under the name of its first element.
    # The engine addresses arrays by base name plus an offset, so the
    # subscript is stripped here.
    IF element count > 1 THEN strip a trailing "[0]" from name

    value_type := float, OR int for integer vectors, OR bool for boolean vectors
    location   := program.location_of(name)

    SELECT declared type
      scalar / 2-, 3-, 4-vector (of any of float, int, bool) ->
          class := the matching vector class
      4x2 matrix -> class := 2x4
      4x3 matrix -> class := 3x4
      4x4 matrix -> class := 4x4
      2x2 or 3x3 matrix -> FAIL WITH "unsupported matrix shape"
      any sampler (1D, 2D, 3D, cube, shadow, multisample) ->
          register_sampler(name, i, location, program, destination)
          CONTINUE          # samplers are not ordinary constants
      otherwise -> FAIL WITH "unsupported uniform type"

    entry := table.find(name)
    IF entry EXISTS THEN
        entry.destination := entry.destination OR destination
        ASSERT entry.type == value_type
    ELSE
        entry := table.add(name, value_type, destination)
    entry.load_for(destination) := { i, class, location, program }

  sort table by name
  RETURN true
```

**Invariants** — only 4-column matrices are modelled. The engine's matrices are always four columns wide and two, three or four rows tall, matching the transform shapes it actually uses (a full transform, a rotation-plus-translation, a two-row texture generator). A square 2x2 or 3x3 in a shader source is a hard failure, which is a deliberate early warning that the shader has drifted from the engine's conventions.

**Notes** — the enumeration *index* is stored as well as the location. For ordinary constants nothing reads it; for samplers it becomes the texture slot, which is the subtle part below.

## Sampler registration (folded into `parse`)

**Contract** — a sampler uniform becomes a table entry of sampler type carrying the sampler binder. Its *index* — not its location — is the texture slot the engine will bind a texture into.

```text
IF the program is monolithic (destination is whole-program) THEN
    slot := enumeration index
ELSE
    # With separable programs each stage enumerates from zero, so the vertex
    # stage's indices would collide with the pixel stage's. The vertex stage's
    # slots are therefore offset past the pixel stage's reserved range.
    slot := enumeration index + (pixel stage ? 0 : vertex-stage base)
```

**Invariants** — when a sampler of the same name is already in the table, every recorded field must match exactly: same slot, same location, same program, same binder. A mismatch aborts. This asserts that the two stages of one pass agree about where a shared sampler lives, which they do only because the pass's programs are compiled from the same source tree with the same macro set.

**Notes** — **The sampler slot is derived from enumeration order, not declared.** That makes the texture binding of every material pass dependent on the order the driver chooses to enumerate uniforms in. It works because a given driver is self-consistent and because the slot is written into the program by the binder rather than assumed — but it means a compiled-program cache entry is only valid for the driver that produced it, which is exactly why the cache key includes the adapter and version strings ([`rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md)). A rebuild should declare sampler slots explicitly in the shader source and delete this inference.

## `sampler_binder`

**Contract** — the producer attached to every sampler constant. When the pass is installed, it writes the sampler's assigned slot into the program's sampler uniform, so that the texture the engine binds at that slot is the one the shader reads. Uses the program-targeted write when separable programs are in play, and the currently-bound-program write otherwise.

**Notes** — this is the only *binder* this backend adds; the rest of the standard binding set is shared and lives in [`Blender_Recorder_StandartBinding.cpp`](../xrRender/Blender_Recorder_StandartBinding.cpp.md). It exists because the slot assignment above is computed at parse time and has to be pushed into the program before the first draw.
