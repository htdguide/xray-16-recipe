# src/xrEngine/xrImage_Resampler.h

> Declares the filter menu and the rescale entry point; the substance is in [`xrImage_Resampler.cpp`](xrImage_Resampler.cpp.md).

**Needs** — [`xrImage_Resampler.cpp`](xrImage_Resampler.cpp.md)
**Used by** — [`dx11r_screenshot.cpp`](../Layers/xrRenderDX11/dx11r_screenshot.cpp.md) · [`xrImage_Resampler.cpp`](xrImage_Resampler.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface described in [`xrImage_Resampler.cpp`](xrImage_Resampler.cpp.md).

Exported units:

- **The filter enumeration** — seven named reconstruction kernels, in order: the default
  cubic ease, box, triangle, bell, B-spline, Lanczos-3, Mitchell. The numbering is
  arbitrary and internal; nothing on disk stores it.
- **`imf_Process`** — rescale a 32-bit four-channel image from one size to another with a
  chosen filter, source and destination supplied by the caller.
