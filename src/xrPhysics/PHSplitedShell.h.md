# src/xrPhysics/PHSplitedShell.h

> The shell a fragment gets when a breakable object comes apart: a normal shell whose spatial footprint is capped and whose collision is static-only.

**Needs** — [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md) · [`PHShell.h`](PHShell.h.md)
**Used by** — [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md) · [`PhysicsShell.cpp`](PhysicsShell.cpp.md)
**Tier floor** — T2: it overrides three shell behaviours; nothing here touches a device or a format.

## Purpose

Declares the surface implemented in [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md).

A fragment of a broken object is *almost* an ordinary shell, and the three places it differs are the
whole content of this type. It is a separate type rather than a flag on the shell because the
differences are in behaviour the shell exposes for overriding, and because a fragment never becomes
an unsplit shell again.

## State

```text
RECORD SplitedShell EXTENDS Shell
  max_aabb_radius : real      # default: unbounded
```

## Exported units

- **`SetMaxAABBRadius`** — caps the radius the shell reports to the spatial index. The base shell
  ignores this; a fragment honours it.
- **`Collide`** — collide against the static world only.
- **`get_spatial_params`** — derive the bounding sphere and box from the shell's collision space,
  then clamp the radius.
- **`DisableObject`** — leave the world's active-object set outright rather than merely sleeping.

Contracts are in [`PHSplitedShell.cpp`](PHSplitedShell.cpp.md).
