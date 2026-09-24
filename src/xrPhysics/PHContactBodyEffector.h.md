# src/xrPhysics/PHContactBodyEffector.h

> Declares the drag a body feels from the medium it is touching — how water,
> mud and snow slow things down without being simulated as fluids.

**Needs** — [`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md) · [`PHBaseBodyEffector.h`](PHBaseBodyEffector.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md) · [`PHElement.cpp`](PHElement.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`Physics.cpp`](Physics.cpp.md) · [`Physics.h`](Physics.h.md)
**Tier floor** — T1: it is attached to a body through the solver's own per-body user slot.

## Purpose

Declares the surface implemented in
[`PHContactBodyEffector.cpp`](PHContactBodyEffector.cpp.md). One body, one contact, one
material: the state is deliberately tiny because an effector lives for one step.

## exported units

- **`Init`** — bind the effector to a body, the contact that created it, and the material
  that contact was against. Computes and stores the material's *resistance* — one minus its
  flotation factor — which is the only property of the material the effector keeps.
- **`Merge`** — fold a second contact against a possibly different material into the same
  effector, so a body touching several surfaces at once gets one drag, not several. Only the
  resistance is merged, by taking the larger.
- **`Apply`** — add the drag force to the body and detach the effector.

**Notes** — `Merge` keeps the *most* resistant of the materials touched, not an average. A
body half in water and half on stone is treated as being in water. That is the conservative
choice: over-damping looks like heavy going, under-damping looks like an object skating
across a pond.
