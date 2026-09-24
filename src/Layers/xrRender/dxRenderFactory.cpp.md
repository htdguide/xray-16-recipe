# src/Layers/xrRender/dxRenderFactory.cpp

> The one object that turns each engine-side request for a "renderer companion" into a concrete instance of this backend's class, and frees it back into this module's allocator.

**Needs** — [`Include/xrRender/RenderFactory.h`](../../Include/xrRender/RenderFactory.h.md) · [`dxRenderFactory.h`](dxRenderFactory.h.md) · [`dxStatGraphRender.h`](dxStatGraphRender.h.md) · [`dxThunderboltRender.h`](dxThunderboltRender.h.md) · [`dxThunderboltDescRender.h`](dxThunderboltDescRender.h.md) · [`dxRainRender.h`](dxRainRender.h.md) · [`dxUIShader.h`](dxUIShader.h.md) · [`dxUISequenceVideoItem.h`](dxUISequenceVideoItem.h.md) · [`dxWallMarkArray.h`](dxWallMarkArray.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxRenderFactory.h`](dxRenderFactory.h.md)
**Tier floor** — T1: the allocation and the matching free must both happen inside this module, because in a dynamically-loaded-renderer build the engine and the renderer can hold different allocators.

## Purpose

Chapter 4 declares a dozen small interfaces, each the renderer-side half of an engine-side object: the engine owns the weather state, this module owns the sky domes; the engine owns a lightning descriptor, this module owns its mesh. [`RenderFactory.h`](../../Include/xrRender/RenderFactory.h.md) is the only door through which the engine may make one. This file is that door's filling for the shared renderer core: for every interface, a creator that returns a fresh instance of the `dx`-prefixed class and a destroyer that releases it.

The file is almost content-free and that is the point — every line is a mechanical pairing. What is load-bearing is the *rule* the pairing encodes and the *set* of pairings that must exist.

## State

```text
RECORD RenderFactory
  # no fields
```

One process-wide instance exists, published into the global environment when a renderer is selected at startup. It is stateless: it holds no registry, no pool, no list of what it has made. Ownership of every product passes to the caller, and the caller is responsible for handing it back.

## `RenderFactory`

**Contract** — For each companion interface `X` in chapter 4 the factory exposes `create_X` returning a freshly allocated implementation, and `destroy_X` taking one back. Creation allocates and never fails softly — an exhausted allocator is a fatal condition, not a `none` return, because every caller immediately dereferences the result. Destruction is idempotent only in the sense that the caller must not pass the same object twice; the factory does not track what it handed out.

**Invariants**

- Every object created here is destroyed here. This is the whole reason the factory exists rather than each call site constructing what it wants: when the renderer is a separately loaded module, the memory came from that module's allocator and must go back to it.
- The destroyer is given the *interface* reference and internally treats it as the concrete type. Nothing checks that the object actually came from this factory; passing a foreign object is undefined. A rebuild in a language with safe downcasts should make this a checked conversion, but must keep the module-affinity rule.
- The create/destroy set must cover *exactly* the interfaces chapter 4 declares. A missing pair is a link failure in the original and must be a compile-time failure in a rebuild too; the factory is the contract that the backend is complete.

```text
FUNCTION create_<X>() -> <X>
  RETURN new dx<X>

FUNCTION destroy_<X>(obj : <X>)
  release obj as dx<X>
```

## Product set

The companions this backend produces, and where each is described:

| Companion | Implementation |
|---|---|
| UI sequence video item | [`dxUISequenceVideoItem.cpp`](dxUISequenceVideoItem.cpp.md) |
| UI shader | [`dxUIShader.cpp`](dxUIShader.cpp.md) |
| Stat graph renderer | [`dxStatGraphRender.cpp`](dxStatGraphRender.cpp.md) |
| Object-space (debug) renderer | `dxObjectSpaceRender` |
| Wallmark array | [`dxWallMarkArray.cpp`](dxWallMarkArray.cpp.md) |
| Flare renderer | `dxFlareRender` |
| Thunderbolt renderer | [`dxThunderboltRender.cpp`](dxThunderboltRender.cpp.md) |
| Thunderbolt descriptor renderer | [`dxThunderboltDescRender.cpp`](dxThunderboltDescRender.cpp.md) |
| Rain renderer | [`dxRainRender.cpp`](dxRainRender.cpp.md) |
| Lens-flare renderer | `dxLensFlareRender` |
| Debug-overlay renderer | `dxImGuiRender` |
| Environment renderer | `dxEnvironmentRender` |
| Environment-descriptor renderer | `dxEnvDescriptorRender` |
| Font renderer | `dxFontRender` |

**Notes**

Two of these are conditional and the conditions are decisions, not accidents:

- The object-space renderer — the debug visualiser for the collision database — exists only in a debug build. A release build must still satisfy the interface *set*, which it does because chapter 4 also declares that pair conditionally. A rebuild that always compiles it pays only the code size.
- Everything except the font renderer is absent in the *editor* configuration, which links the same renderer core against a different host. The font renderer is unconditional because text is the one thing every configuration draws.

The whole family is generated in the original by a textual macro over the interface name. That is a C++ convenience; what survives is the naming convention it enforces — interface `IX`, implementation `dxX`, creator `CreateX`, destroyer `DestroyX` — which is what lets a rebuilder verify completeness by inspection.
