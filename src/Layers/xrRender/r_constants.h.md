# src/Layers/xrRender/r_constants.h

> The reflected shader interface: what a shader constant is, where in which stage it lives, and how a single integer encodes every destination it can occupy.

**Needs** — [`r_constants.cpp`](r_constants.cpp.md) · [`xrCore/xr_resource.h`](../../xrCore/xr_resource.h.md) · [`r_constants_cache.h`](r_constants_cache.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`Blender_Recorder_R2.cpp`](Blender_Recorder_R2.cpp.md) · [`DetailManager.h`](DetailManager.h.md) · [`R_Backend_xform.cpp`](R_Backend_xform.cpp.md) · [`R_Backend_xform.h`](R_Backend_xform.h.md) · [`SH_Atomic.cpp`](SH_Atomic.cpp.md) · [`SH_Atomic.h`](SH_Atomic.h.md) · [`Shader.cpp`](Shader.cpp.md) · [`Shader.h`](Shader.h.md) · [`ShaderResourceTraits.h`](ShaderResourceTraits.h.md) · [`TextureDescrManager.cpp`](TextureDescrManager.cpp.md) · [`TextureDescrManager.h`](TextureDescrManager.h.md) · [`dxUISequenceVideoItem.cpp`](dxUISequenceVideoItem.cpp.md) · _and 11 more_
**Tier floor** — T1: the destination word is a hand-packed bit field whose layout is relied upon by the backends' binding code, and a constant's binding record is looked up per draw.

## Purpose

Shaders ship as *source* with the game data (§5 of the requirements), so the engine cannot know a shader's constants until it has compiled it. After compiling, it asks the compiler what the shader declared and builds a **constant table**: for each named constant, what kind it is, which stages use it, and where in each stage it lives.

Everything in the renderer that sets a shader value — the frame's matrices, a light's colour, an object's hemisphere term, a material's animated texture offset — goes through this table. This header declares its vocabulary; [`r_constants.cpp`](r_constants.cpp.md) implements the table's operations.

## Kinds and shapes

```text
ENUM ConstantKind { float, integer, boolean, sampler, texture, unordered_access }
ENUM ConstantShape
  scalar_or_vector1, vector4, vector3, vector2,
  matrix_2x4, matrix_3x4, matrix_4x4,           # all stored transposed
  array_of_vector4, array_of_matrix_3x4, array_of_matrix_4x4
```

Two notes that matter for a rebuild:

- Every matrix shape is **transposed** on the way in. The shader declares row-major and the constant storage is column-major, or the reverse depending on backend; either way the engine transposes once at the set rather than letting the compiler emit a transpose per use.
- A *sampler* and a *texture* are the same constant on older devices and different resources on newer ones, which is why both kinds exist and why the sampler binding is stored separately from the stage bindings. A rebuild targeting only modern devices keeps them separate and deletes the merged case.

## The destination word

A constant may appear in several stages at once, and in each stage may live in a particular constant buffer. All of that is packed into one integer:

```text
bits 0..7   which stages use this constant:
              pixel, vertex, sampler, geometry, hull, domain, compute,
              and a "all stages" bit used only by the backend whose programs
              are linked as a unit
bits 8..11  the geometry stage's constant buffer index   (0..14)
bits 12..15 the vertex stage's constant buffer index
bits 16..19 the pixel stage's constant buffer index
bits 20..23 the hull stage's constant buffer index
bits 24..27 the domain stage's constant buffer index
bits 28..31 the compute stage's constant buffer index
```

**Invariants**

- The low byte's bit order is relied upon by code that does arithmetic on it rather than testing named bits. The header says so explicitly; a rebuild that reorders the stages breaks binding silently.
- A buffer index field is four bits and admits values zero to fourteen. Fifteen is unused, which leaves room for a "no buffer" marker that is never written.
- The whole packing exists so that a constant's destination fits in one word alongside its bindings, because the table is scanned per draw. A rebuild with cheap aggregates should use a record of fields and lose nothing but the bit arithmetic.

There is a second, smaller packing for identifying a constant buffer itself — a four-bit index plus a three-bit stage tag — used when buffers are bound rather than constants.

## Records

```text
RECORD Binding                     # where one constant lives in one stage
  index    : int (16-bit)          # the register or offset
  shape    : int (16-bit)          # one of the shapes above
  # on the backend whose programs are linked as a unit, also:
  location : int                   # the resolved uniform location
  program  : int                   # which linked program it belongs to

RECORD Constant
  name        : text               # the name written in the shader source — FROZEN
  kind        : ConstantKind
  destination : int (32-bit)       # the packed word above
  bindings    : one Binding per stage, plus one for the sampler slot
  setter      : optional<ConstantSetter>

RECORD ConstantTable
  entries              : list<Constant>   # sorted by name
  legacy_compatibility : bool
  buffers              : per render context, list<(usage, Buffer)>
```

Invariants:

- **The names are frozen.** They are the identifiers written in the shipped shader sources; the engine finds a constant by string. Every well-known name — the world matrix, the base texture sampler, the fog parameters — is part of the data contract and cannot be renamed.
- `entries` is kept sorted by name so lookup by string can binary-search. Every mutation that appends must re-sort.
- `legacy_compatibility` records that the table came from a shader written for the oldest device, where two constants may share a name and be distinguished only by kind. It changes how lookup and merge behave; see [`r_constants.cpp`](r_constants.cpp.md).
- The constant buffers are held **per render context**, because a buffer is written and bound by whichever context is drawing and several contexts draw concurrently.

## `ConstantSetter`

**Contract** — An abstract hook: given a command list and a constant, write that constant's current value. This is how *automatic* constants work — the ones the shader names but the material never sets, such as the current time, the camera position, or the sun's direction. Binding a pass walks its table and invokes each constant's setter.

That the setter lives on the constant rather than in a central table is the design decision: a shader that names a well-known constant gets it filled without anyone registering anything, which is what lets shipped shaders use engine values the engine does not know they want.

## `ConstantTable` operations

Declared here, described in [`r_constants.cpp`](r_constants.cpp.md): `clear`, `parse` (build from a compiler's reflection of one stage), `merge` (fold another stage's table into this one), the two `get` lookups, and `equal`.

**Notes** — The equality test on a constant compares its name, kind, destination, every binding and its setter — everything. It exists so that two independently compiled shader elements can be recognised as interchangeable and the backend can skip a state change between them; that is the same equality the opaque pass sort in [`r__dsgraph_render.cpp`](r__dsgraph_render.cpp.md) relies on.
