# src/xrPhysics/PhysicsShellAnimator.h

> Declares the animator that drags a shell's bodies toward an animation's pose.

**Needs** — [`PhysicsShell.h`](PhysicsShell.h.md) · [`PhysicsShellAnimatorBoneData.h`](PhysicsShellAnimatorBoneData.h.md) · [`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md)
**Used by** — [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md) · [`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md) · [`PhysicsShellAnimatorBoneData.h`](PhysicsShellAnimatorBoneData.h.md)
**Tier floor** — T2: a list of controlled bodies and a per-frame update.

## Purpose

Declares the surface implemented in
[`PhysicsShellAnimator.cpp`](PhysicsShellAnimator.cpp.md). An animator is created for a shell
whose spawn configuration carries an `animated_object` section, and lives as long as the shell
does.

Exported units:

- **the animator** — constructed from a shell plus a configuration section, holds the list of
  controlled bodies and the shell's starting transform, and is destroyed by releasing its
  constraints.
- **`OnFrame`** — the per-frame step that re-aims every controlled body at the pose animation
  currently says it should have.
