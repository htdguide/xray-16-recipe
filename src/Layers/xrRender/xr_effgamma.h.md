# src/Layers/xrRender/xr_effgamma.h

> Declares the gamma/brightness/contrast holder every renderer backend embeds.

**Needs** — [`xr_effgamma.cpp`](xr_effgamma.cpp.md)
**Used by** — [`D3DXRenderBase.h`](D3DXRenderBase.h.md) · [`xr_effgamma.cpp`](xr_effgamma.cpp.md)
**Tier floor** — T2: four numbers and their setters.

## Purpose

Declares the surface implemented in [`xr_effgamma.cpp`](xr_effgamma.cpp.md), where the ramp
curve and the two installation paths live. The type is embedded by value in the shared
renderer base ([`D3DXRenderBase.h`](D3DXRenderBase.h.md)), which is how the engine's three
picture-setting entry points reach it.

## Exported units

- **`CGammaControl`** — the holder. Constructed neutral: gamma, brightness and contrast at
  1 and balance at white.
- **`Gamma`**, **`Brightness`**, **`Contrast`** — set one scalar. Storing only; nothing is
  installed until `Update` is called, which is why the engine sets all three and then
  updates once.
- **`Balance`** — set the per-channel multiplier, in two spellings. No caller exists.
- **`GetIP`** — read all four back. No caller exists.
- **`Update`** — build the ramp and install it on the display.

**Notes** — The class declares two overloads of its private ramp generator, one per
installation target, and one of them exists only where the display-output path is compiled
in. That conditional shape is a C++ packaging concern; the decision underneath it — two
targets, one curve — belongs to the implementation twin.
