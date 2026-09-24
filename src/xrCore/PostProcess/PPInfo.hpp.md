# src/xrCore/PostProcess/PPInfo.hpp

> The complete set of screen-effect parameters the game can push onto a frame, and the rules for combining two of them.

**Needs** — [`PPInfo.cpp`](PPInfo.cpp.md) · [`_color.h`](../_color.h.md) · [`xrstring.h`](../xrstring.h.md)
**Used by** — [`r2_rendertarget_phase_PP.cpp`](../../Layers/xrRender_R2/r2_rendertarget_phase_PP.cpp.md) · [`PPInfo.cpp`](PPInfo.cpp.md) · [`PostProcess.cpp`](PostProcess.cpp.md) · [`PostProcess.hpp`](PostProcess.hpp.md) · [`CameraManager.h`](../../xrEngine/CameraManager.h.md) · [`EffectorPP.h`](../../xrEngine/EffectorPP.h.md) · [`script_effector.h`](../../xrGame/script_effector.h.md) · [`script_effector_script.cpp`](../../xrGame/script_effector_script.cpp.md)
**Tier floor** — T2: a parameter block and arithmetic on it. It is handed to the renderer as values, not as a buffer layout.

## Purpose

Declares the record whose combination rules live in [`PPInfo.cpp`](PPInfo.cpp.md). This is the *interface* between the game and the renderer's post-processing: a wound, a drug, a radiation field and a psi attack each contribute one of these, the game adds them together, and the renderer reads one combined block per frame.

The split between this header and its implementation is unusually thin — the record and the inline colour and pair helpers are here, only the combination rules are next door. A rebuild should treat the two files as one unit.

## Exported units

- **`SPPInfo`** — the parameter block. Its fields, with the neutral value each takes when no effect is active:
  - `blur` (0) and `gray` (0) — screen blur amount, and desaturation amount.
  - `duality.h`, `duality.v` (0, 0) — horizontal and vertical ghosting offsets; the "double vision" effect.
  - `noise.intensity` (0), `noise.grain` (1), `noise.fps` (10) — film-grain strength, grain size, and how many times a second the grain pattern is re-rolled.
  - `color_base` (0.5, 0.5, 0.5) — the mid-point the contrast curve pivots around. Note the neutral value is a half, not zero, because it is a *pivot*, not an offset.
  - `color_gray` (0.333, 0.333, 0.333) — the per-channel weights used to compute the grey the image is pulled toward. The neutral value is a flat average, not a perceptual luminance weighting.
  - `color_add` (0, 0, 0) — a colour added after everything else; how a red flash is done.
  - `cm_influence` (0), `cm_interpolate` (0), `cm_tex1`, `cm_tex2` — colour-mapping: up to two named lookup textures, how strongly the mapping applies, and where between the two to sample.
- **`SPPInfo.SColor`** — a three-channel colour with add, subtract and set, convertible to a packed 32-bit colour with a zero alpha and to a three-component vector. The vector conversion is a reinterpretation of the same three floats, which is only legal because the layouts agree — a rebuild should convert rather than alias.
- **`SPPInfo.SDuality`**, **`SPPInfo.SNoise`** — the two named field groups above.
- **`add`**, **`sub`**, **`lerp`**, **`normalize`**, **`validate`** — the combination rules; see [`PPInfo.cpp`](PPInfo.cpp.md).
