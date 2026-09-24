# src/xrEngine/phdebug.cpp

> Holds the process-wide handle to the physics debug renderer, which is empty unless a debug build installs one.

**Needs** — [`IPHdebug.h`](IPHdebug.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one nullable handle.

## Purpose

The physics module wants to draw its own diagnostics — contact points, joint frames,
collision shapes — but it must not depend on the renderer. So the renderer, when it is
willing, installs an implementation here and the physics module draws through it if it is
present.

This file exists solely to give that handle a home in a module both sides already link
against. It is a one-slot service locator, the same pattern as the global environment
struct in the preface and with the same advice: a rebuild should inject the debug renderer
rather than reach for a global.

## State

```text
physics_debug_renderer : optional<PhysicsDebugRenderer>   # none unless installed
```

**Invariant** — every use site checks for absence. The handle is empty for the entire life
of a shipping build.
