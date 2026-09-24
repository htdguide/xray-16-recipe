# src/Layers/xrRender/FBasicVisual.cpp

> The model header every model file starts with: version, type tag, bounds and material — and the rule that a duplicated model shares its original's geometry.

**Needs** — [`FBasicVisual.h`](FBasicVisual.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`ResourceManager.h`](ResourceManager.h.md)
**Used by** — reached through its declarations in [`FBasicVisual.h`](FBasicVisual.h.md); callers name that, not this file.
**Tier floor** — T1: the header chunk is read as a byte image from a shipped model file.

## Purpose

The base of the model hierarchy, and it does exactly two things: read the shared header out of a model file, and define what copying a model means. Both are short and both are load-bearing for every model type above.

## The model header — frozen

```text
RECORD ModelHeader              # the first chunk of every model file
  format_version : int (8-bit)  # a single frozen value; a mismatch is fatal
  type           : int (8-bit)  # which model class this file is
  shader_id      : int (16-bit) # index into the LEVEL's shared material table, or zero
  bounding_box   : box
  bounding_sphere: sphere
```

**Invariants**

- **The version is checked, not negotiated.** There is one accepted value and a file carrying any other aborts the load with a message. The format shipped once and never changed; a rebuild should do the same rather than attempt a migration for a version that does not exist.
- **The type tag chooses the class**, and it is read here, by the base, before the subclass exists. The model pool reads the header, dispatches on the type to construct the right class, and then hands the same stream back to that class to finish loading. This is why the base reads a field it does not itself use.
- **A model's material is named two different ways**, and both must be supported because both appear in shipped data. A *level* model carries a 16-bit index into the level's shared material table, resolved through the renderer. A *standalone* model carries a later chunk with the material template name and the texture name as strings. When both are present the second wins, because it is read second — and that is not defensive, it is how a level model with an overridden material is expressed.
- A shader identifier of zero means "no material", which is legal: a container model that only holds children has geometry nowhere and a material nowhere.
- The bounds are read from the file, unlike the detail models', because a model may have geometry the engine never sees (a collision proxy, a progressive mesh's full detail) and the authored bounds are authoritative.

## `copy(source)`

**Contract** — makes this model a duplicate of another. Copies the type tag, the material reference, and the visibility record. Does **not** copy geometry — see below.

**Invariants**

- **A duplicated model shares its original's geometry and material.** This is the whole point of the operation: the game creates hundreds of instances of one model and they must all draw out of one vertex buffer. What a duplicate gets of its own is the fields the renderer writes per instance — chiefly the visibility record, which carries per-instance occlusion state.
- The copy is *shallow by design and by field*. Each derived type adds its own copy step, and each decides field by field what is shared and what is per instance. A rebuild that makes copying structural (a deep clone, a value copy) breaks the sharing that makes the model pool work; it should instead separate the shared *asset* from the per-instance *record*, which is what this copy is manually approximating.

## The mesh record's lifetime

**Contract** — destroying a mesh releases its geometry declaration and drops one reference on each of its two buffers.

**Notes** — The buffers are reference counted precisely because of the sharing described above: a level's static geometry lives in a handful of large buffers and every model in the level is a slice of one. The last model referencing a buffer releases it. That is a decision a rebuild must make explicitly — the alternative, one buffer per model, costs a bind per draw and is why the shipped engine does not do it.
