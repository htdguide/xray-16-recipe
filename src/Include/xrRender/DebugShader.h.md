# src/Include/xrRender/DebugShader.h

> Names the debug layer's material type: it is the same handle the UI uses.

**Needs** — [`FactoryPtr.h`](FactoryPtr.h.md) · [`UIShader.h`](UIShader.h.md)
**Used by** — [`DebugRender.h`](DebugRender.h.md) · [`dxDebugRender.cpp`](../../Layers/xrRender/dxDebugRender.cpp.md) · [`LevelGraphDebugRender.hpp`](../../xrGame/LevelGraphDebugRender.hpp.md)
**Tier floor** — T2: a naming decision, no behaviour.

## Purpose

The debug renderer needs materials. Rather than defining its own material interface, it declares that a debug material *is* a [UI material handle](UIShader.h.md) — same interface, same factory, same ownership policy.

The whole file is that one statement. It exists so that [`DebugRender.h`](DebugRender.h.md) can name the type without pulling the UI header into the debug layer's vocabulary, and so the decision is reversible in one place.

## State

`Stateless.`

## `debug_shader`

**Contract** — an alias for a factory-owned [UI material handle](UIShader.h.md). Everything about creation, ownership, copying and destruction is that type's.

**Notes** — The aliasing is the decision worth recording: **debug drawing and UI drawing want exactly the same thing from a material** — a named pass chain plus a texture, resolved from data, with no programmatic state to set. A rebuild that gives them separate material types will find the second one is a copy of the first.
