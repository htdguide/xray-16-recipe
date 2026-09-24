# src/Layers/xrRender/ResourceManager_Resources.cpp

> The interning primitives: one create/delete pair per resource kind, implementing the two rules — intern by name, or intern by value — that everything else in the registry is built from.

**Needs** — [`ResourceManager.h`](ResourceManager.h.md) · [`Shader.h`](Shader.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [`SH_Matrix.h`](SH_Matrix.h.md) · [`SH_Constant.h`](SH_Constant.h.md) · [`SH_RT.h`](SH_RT.h.md) · [`SH_Atomic.h`](SH_Atomic.h.md) · [`tss.h`](tss.h.md) · [`Texture.cpp`](Texture.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: several of these creators allocate device objects, and the by-value interning depends on a bit-level equality over records whose layout is fixed.

## Purpose

Everything the renderer draws with is shared. A level's few thousand materials resolve to a few hundred distinct pass lists, a few dozen distinct state blocks and one texture per distinct name. This file is where that sharing is implemented, and it is worth reading as *two* algorithms repeated a dozen times rather than as a dozen functions.

It is a separate file only by topic. Its content is mechanical; what is load-bearing is the pair of rules and the handful of places that deviate from them.

## State

`Stateless`; it is the mutator of the registry described in [`ResourceManager.h`](ResourceManager.h.md).

## The two interning rules

```text
# Rule A — intern by NAME.  Used for textures, animators and render targets:
# things whose identity IS their name, and whose content is loaded from it.
FUNCTION create_by_name(map, name, build) -> optional<Resource>
  IF name is a reserved "nothing" spelling
    RETURN none
  IF map has name
    RETURN map[name]
  resource = build()
  mark it registered
  map[resource.set_name(name)] = resource     # the key is the resource's OWN name string
  RETURN resource

FUNCTION delete_by_name(map, resource)
  IF resource is not marked registered
    RETURN                                    # a prototype the registry never owned
  IF map has resource's name
    erase it; RETURN
  report "failed to find <kind> '<name>'"      # log, do not raise: teardown must finish

# Rule B — intern by VALUE.  Used for everything compiled: state blocks, declarations,
# geometries, constant tables, the three resource lists, passes, elements, shaders.
FUNCTION create_by_value(list, prototype) -> optional<Resource>
  IF prototype is empty by that kind's definition
    RETURN none                               # empty and absent must not both exist
  FOR EACH existing IN list
    IF existing equals prototype
      RETURN existing
  resource = list.append(a copy of prototype)
  mark it registered
  RETURN resource

FUNCTION delete_by_value(list, resource)
  IF resource is not marked registered
    RETURN
  IF remove resource from list
    RETURN
  report "failed to find compiled <kind>"
```

**Invariants**

- **The map key is the resource's own name storage**, assigned in the same expression that inserts it. Nothing may rename a registered resource: the key would still point at the old bytes, or at freed ones.
- **The registered flag is what makes rule B safe.** Every by-value creator is handed a prototype built on the caller's stack. That prototype is destroyed when the caller returns, and its destructor calls the delete path. The flag is absent on the prototype, so the delete is a no-op; it is present only on the copy the registry holds. A rebuild that hands ownership rather than copying does not need the flag — but must then not destroy the prototype.
- **A failed delete logs and continues.** Every one of these paths ends in a report rather than an abort, because they run during teardown and during device loss, where raising loses the diagnostic.
- Both rules are linear scans. The by-value lists reach a few thousand entries at their largest (passes, elements), and creation happens at level load, not per frame.

## `_CreateTexture` · `_DeleteTexture`

**Contract** — rule A over the texture map, with two extra steps. The name is normalized the same way a material's texture list is (lowercase, known extensions stripped). The reserved spelling is `null` — exactly that, case-sensitively — which yields nothing. After registering, the texture's header is read immediately (`Preload`) and its pixels are uploaded straight away *unless* deferred loading is on, in which case the upload waits for the batch (see [`ResourceManager.cpp`](ResourceManager.cpp.md)).

**Notes** — The split between preload and load is the load-time budget: the header tells the material system the dimensions and format it needs to compile against, and that is cheap; the decode and upload are expensive and are batched. A rebuild wanting a single-phase texture load must still answer "what does the material compiler need before the pixels exist".

## `simplify_texture`

**Contract** — a debug-build filter, active only under a named command-line switch, that replaces most texture names with the placeholder. Seven categories are exempt by path prefix or name fragment: the writable user root, the user-interface directory, lightmaps, and the actor, effect, glow and map directories.

**Notes** — This is a level-designer tool: it strips the art so that geometry and lighting can be judged without it, while keeping the textures that carry *information* rather than appearance. The exemption list is the interesting part — it is a statement about which of the game's textures are functional. A rebuild may drop the whole thing; if it keeps it, the list is the content.

## `_CreateMatrix` · `_CreateConstant` and their deletes

**Contract** — rule A over the animator maps. The reserved spelling is `$null`, matched case-insensitively. Both are created with **one reference already taken**, which the registry keeps: these are loaded eagerly from the material library and must outlive any material that uses them.

**Notes** — Matrix animators drive texture coordinate transforms over time (scrolling, rotating); constant animators drive scalar and colour constants, and exist only for the oldest renderer generation, where a material could not compute them itself. Both are *definitions*, not instances: one animator is shared by every material that names it, and it is evaluated once per frame, not once per material.

## `_CreateRT` · `_DeleteRT`

**Contract** — rule A over the render-target map, keyed by name, with dimensions, format, sample count and slice count as creation parameters. Asserts a non-empty name and non-zero dimensions. The device object is created immediately if the device is ready, and left uncreated otherwise — the reset path (see [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md)) will create it later.

**Invariants** — the parameters are *not* part of the identity. Asking for an existing name with different dimensions returns the existing target and silently ignores the new parameters. Render-target names are a small fixed set owned by the backends, so the collision cannot arise in practice; a rebuild should still either include the parameters in the key or assert they match.

## `_CreateState` · `_DeleteState`

**Contract** — rule B over state blocks, compared by their *recorded settings* rather than by the resulting device object. On a miss, the recording is replayed into a fresh device object and both are kept: the object to bind, the recording to rebuild it after a device loss.

**Invariants** — the recording is the source of truth and must be retained for the lifetime of the block. This is what makes the reset path possible.

## `_CreateConstantTable` · `_DeleteConstantTable`

**Contract** — rule B over constant tables, with *empty interns to nothing*. A constant table maps a program's named constants to their binding locations; a program with none gets no table, and callers test for its absence.

## `CreateGeom` · `DeleteGeom`

**Contract** — rule B over geometry records, but the identity is a four-part tuple compared field by field rather than a value equality: declaration, vertex buffer, index buffer and vertex stride. The declaration is itself interned first, so comparing declarations is a pointer comparison. The stride is computed from the declaration at stream zero and cached in the record.

```text
FUNCTION create_geometry(declaration, vertex_buffer, index_buffer) -> Geometry
  interned_decl = intern(declaration)
  stride = byte size of one vertex in stream 0 of declaration
  FOR EACH existing geometry
    IF all four of (declaration, vertex buffer, index buffer, stride) match
      RETURN it
  RETURN a new registered geometry with those four fields
```

**Notes**

- The stride is stored even though it is derivable, because it is read on every draw call. Caching a derived value in the hot path is the decision; a rebuild should make the same one.
- A second overload accepts the legacy packed vertex-format word instead of a declaration and expands it to one first. That word is a fixed bit encoding from the graphics API of the era, and it survives because shipped data and older engine code express vertex layouts with it. A rebuild needs the expansion only if it keeps that encoding at its boundaries.
- Geometry records are the objects the reset path repairs when the dynamic buffers are rebuilt, which is why the buffer handles are part of the identity rather than resolved per draw.

## `_CreateTextureList` · `_CreateMatrixList` · `_CreateConstantList` and their deletes

**Contract** — rule B over the three per-pass resource lists, each with its own notion of "empty".

- A **texture list** is a list of (stage, texture) pairs. It is **sorted by stage before comparison**, so that two materials binding the same textures to the same stages in a different authoring order intern to one list. It has no empty case: a pass with no textures still gets a list.
- A **matrix list** and a **constant list** are small fixed-capacity arrays of slots. Both intern to *nothing* when every slot is empty, which is the common case — most materials animate nothing.

**Invariants** — the sort is destructive: the caller's prototype is reordered in place. That is harmless because the prototype is discarded, and load-bearing because the comparison assumes sorted order on both sides.

## `_DeletePass` · `_DeleteDecl` · `_DeleteElement`

**Contract** — rule B deletes. Their creators live elsewhere: passes are created by the backends (the pass is where the device programs are bound), declarations by the backends, elements by [`ResourceManager.cpp`](ResourceManager.cpp.md). That the deletes are here and the creates are not is an accident of how the file grew.

## `DBG_VerifyTextures` · `DBG_VerifyGeoms`

**Contract** — debug-build assertions that each map key still equals its value's name. `DBG_VerifyGeoms` is entirely commented out; it used to verify that a geometry's cached stride still matched its declaration's computed one, which is the invariant most likely to rot.

**Notes** — The disabled check is worth reviving in a rebuild: the cached stride is the one derived value in the registry, and nothing else notices when it drifts.
