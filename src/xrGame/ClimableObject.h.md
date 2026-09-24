# src/xrGame/ClimableObject.h

> Declares the ladder implemented in [`ClimableObject.cpp`](ClimableObject.cpp.md) and the geometric questions it answers for a climbing character.

**Needs** — [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`xrPhysics/IClimableObject.h`](../xrPhysics/IClimableObject.h.md)
**Used by** — [`ClimableObject.cpp`](ClimableObject.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the ladder: an oriented box, a derived three-vector frame, a surface material,
and a static collision registration. It implements the physics layer's climbable
interface, which is how a character's movement code reaches it without knowing it is a
game object. Substance in [`ClimableObject.cpp`](ClimableObject.cpp.md).

Exported units:

- `CClimableObject` — the ladder entity.
- `net_Spawn`, `net_Destroy` — derive the frame from the authored box; release the
  collision volume.
- `Axis`, `Side`, `Norm` — the frame, as scaled (non-unit) vectors.
- `DDAxis`, `DDSide`, `DDNorm` — the same, decomposed into magnitude and unit direction.
- `LowerPoint`, `UpperPoint` — the mount and dismount points, offset onto the climbing
  face.
- `DToAxis`, `DDToAxis`, `POnAxis` — the character's offset from the axis line.
- `DSideToAxis`, `DDSideToAxis` — that offset resolved across the face; the `DD` form is
  unsigned and always points inward.
- `DToPlain`, `DDToPlain` — that offset resolved out of the face.
- `AxDistToUpperP`, `AxDistToLowerP` — signed distances to each end, both positive while
  between them.
- `InRange`, `InTouch`, `BeforeLadder` — within the vertical extent; actually against the
  face; on the front side.
- `Material` — the surface material, for footsteps.
- `ObjectContactCallback` — the physics contact filter that makes a ladder solid from the
  front and transparent from behind.
- `Center`, `Radius` — cull bounds.
- `UsedAI_Locations`, `register_schedule` — both false: a ladder occupies no navigation
  position and never updates.
- `DefineClimbState` — an unused extension point.
