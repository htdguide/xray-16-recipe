# src/Layers/xrRender/SH_Atomic.cpp

> The indivisible device-owned resources a pass is built from — one compiled program per shader stage, a state block, a vertex declaration — and the rule that each removes itself from the registry as it dies.

**Needs** — [`SH_Atomic.h`](SH_Atomic.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`r_constants.h`](r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`SH_Atomic.h`](SH_Atomic.h.md)
**Tier floor** — T1: every record here is a handle to device memory, and the order in which the handles are released is observable.

## Purpose

A pass binds a small number of things that the device treats as opaque units. This file defines them and nothing else: they have no behaviour, only identity, a device handle and — for programs — the table of named constants discovered when they were compiled. They are called *atomic* because they are the leaves of the material tree; a blender compiles down to references to these, and two materials that reach the same program share it.

## State

```text
RECORD Program                      # one per shader stage: vertex, pixel, geometry, hull, domain, compute
  name      : text                  # the registry key: the source file stem plus its macro set
  handle    : device program
  constants : ConstantTable         # named constants discovered by reflecting the compiled program

RECORD InputSignature               # newest backend only
  blob : opaque bytes               # the compiled vertex program's input description

RECORD StateBlock
  handle     : device state object
  state_code : list<state assignment>   # the state as recorded by the blender, kept for comparison

RECORD VertexDeclaration
  elements       : list<VertexElement>  # the common, backend-neutral description
  native         : device declaration   # on the newest backend: one input layout per input signature
```

**Invariants**

- A program's name is the cache key and is *not* the file name: it is the file stem with the macro set that was compiled into it appended. Two compilations of the same source with different macros are different programs, and must be, because the shipped shader sources branch on macros heavily.
- Every program carries its own constant table. A pass's merged table is the union of its programs'; the per-program table is what makes that union computable without re-reflecting.
- A state block keeps the *recorded* state assignments alongside the device object. The device object cannot be read back, and two blenders producing identical state must be detected as identical — so the recording is the comparison key and the device object is its cached compilation.
- On the newest backend a vertex declaration is not a single object: the device requires one input layout per (declaration, vertex-program-signature) pair, so the declaration holds a map from signature to layout, filled lazily as programs are bound against it. The backend-neutral element list is kept regardless, because it is what the model formats describe and what comparison uses.
- **A registered resource removes itself from the registry when its last reference drops.** The registry stores raw pointers it does not own; self-removal is the only thing keeping it free of dangling entries.

## `Program` destruction — one idea, six spellings

**Contract** — release the device program and remove the record from the registry map for its stage.

**Notes**

- The six near-identical destructors are incidental: one behaviour, written once per stage because each stage has its own registry map and its own device release call. A rebuild parameterizes by stage and writes it once.
- The *order* between the two halves is not consistent in the source — some stages unregister first, some release first — and nothing depends on it, because the registry holds no reference. A rebuild should pick one order and keep it, since a registry that did hold a reference would make the order matter.
- On the OpenGL backend a "program" is either a separable program object or a raw shader object, depending on a device capability, and the release call differs. This is the clearest small example of the graphics seam leaking: the record cannot say what it holds without asking the device what generation it is. A rebuild should make the handle a device-owned opaque value with a device-owned release, so the record never branches.

## `StateBlock` destruction

**Contract** — release the device state object and unregister. Identical in shape to the programs.

## `VertexDeclaration` destruction

**Contract** — unregister, then release every device layout the declaration accumulated (one per vertex-program signature on the newest backend, one object on the others).

**Notes** — This destructor unregisters *before* releasing, while the program destructors mostly do the reverse. Again nothing depends on it; it is noted only so a rebuilder does not go looking for a reason.

## `InputSignature`

**Contract** — newest backend only: holds the compiled vertex program's input description, adopting a reference to it on construction and releasing it on destruction. It exists as a separate reference-counted record because *many* input layouts key off *one* signature, and the signature must outlive the program that produced it.

**Notes** — This is a pure consequence of one graphics API's model and has no counterpart elsewhere. What survives a rebuild is the requirement behind it: the vertex layout that the device actually binds depends on both the mesh's declaration and the vertex program's expected inputs, so it cannot be built until both are known, and it must be cached against the pair.
