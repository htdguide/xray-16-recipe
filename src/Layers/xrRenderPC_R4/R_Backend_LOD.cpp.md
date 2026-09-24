# src/Layers/xrRenderPC_R4/R_Backend_LOD.cpp

> Turns a visual's continuous level-of-detail scalar into the integer subdivision factor the tessellation stage wants, and pushes it as a named constant — but only when the bound program asked for one.

**Needs** — [`R_Backend_LOD.h`](R_Backend_LOD.h.md) · [`../xrRender/R_Backend.h`](../xrRender/R_Backend.h.md) · [`../xrRender/r_constants.h`](../xrRender/r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend_LOD.h`](R_Backend_LOD.h.md)
**Tier floor** — T2: one clamp and one write into a constant block; it sits in a T1 module only because of what it writes into.

## Purpose

This backend is the only one that ships a tessellation path, so it is the only one that needs to tell a program *how finely to subdivide*. The scene graph already computes a per-visual level-of-detail scalar for its own purposes (which mesh variant to draw, how to fade an impostor); this file is the adapter that maps that same scalar onto a subdivision factor and delivers it through the ordinary by-name constant channel.

It exists as its own file because it is the one piece of per-draw state the shared draw path in chapter 18 pushes *conditionally on the backend*, and because the mapping curve below is a tuning decision that deserves to be findable.

## State

```text
RECORD LodChannel
  constant : optional<handle>   # the bound program's LOD constant, or none
  backend  : reference          # the submission context this channel writes through
# Invariant: `constant` is none until the material system binds one, and is
# cleared again whenever the program changes. A push with no constant is a no-op,
# not an error: most programs have no tessellation and declare nothing.
```

## `set_LOD(handle)` — the binding half

**Contract** — memoizes the constant this channel will write to. Called once per pass by the material system's by-name binding: a pass that declares a constant under the LOD name gets this channel attached to it, a pass that does not never calls here. See [`../xrRenderDX11/dx11r_constants.cpp`](../xrRenderDX11/dx11r_constants.cpp.md) for how the name is resolved to a handle in the first place.

## `set_LOD(real)` — the push

**Contract** — converts the scalar to a subdivision factor and writes it. Does nothing when no constant is bound. Never blocks, never allocates.

```text
FUNCTION push_lod(lod)                 # lod is in [0, 1]; see below
  IF constant IS none THEN RETURN
  factor := clamp(ceil(lod^5 * 8), 1, 7)
  backend.set_constant(constant, factor)
```

**Invariants** — the incoming scalar is already normalized: the scene graph derives it from the visual's screen-space area, rescaled between a "start subdividing" and a "stop subdividing" threshold and square-rooted, so 0 means "at or beyond the far threshold" and 1 means "at or inside the near one". The channel may therefore assume the range and does not re-clamp the input.

**Notes** — the curve, not the constants, is the content. Raising a value already in [0, 1] to the **fifth** power before scaling makes the mapping violently non-linear: something at three-quarters of the band gets factor 2, something at nine-tenths gets factor 5, and only the last sliver of the band reaches the ceiling. Subdivision therefore costs nothing at all for almost everything on screen and everything for the handful of objects in the player's face — which is the only way a per-draw subdivision factor is affordable at all.

The ceiling is **7**, not 8, even though the scale factor is 8: the top of the range clamps down. The floor of 1 means "do not subdivide", so the factor is never zero and the program needs no special case. A rebuild is free to pick a different curve, but should keep the two properties that matter: the factor is a small integer, and it is 1 across most of the visible range.

**Notes** — the scalar is pushed for *every* drawn visual regardless of whether the bound pass tessellates; the cost of that is one branch on a null handle, which is cheaper than asking the pass. A rebuild with an explicit per-pass parameter list would simply not include this channel in passes that do not declare it.
