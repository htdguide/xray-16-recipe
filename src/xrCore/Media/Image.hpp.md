# src/xrCore/Media/Image.hpp

> Declares the engine's one in-memory raster image: pixels, a format tag, and the two containers it can be written to or read from.

**Needs** — [`Image.cpp`](Image.cpp.md) · [`ImageJPEG.cpp`](ImageJPEG.cpp.md) · [`FS.h`](../FS.h.md)
**Used by** — [`dx11r_screenshot.cpp`](../../Layers/xrRenderDX11/dx11r_screenshot.cpp.md) · [`Image.cpp`](Image.cpp.md) · [`ImageJPEG.cpp`](ImageJPEG.cpp.md) · [`UIServerInfo.cpp`](../../xrGame/ui/UIServerInfo.cpp.md)
**Tier floor** — T1: the header it writes is a byte-exact struct and the pixel buffer is handed to a foreign codec as raw memory.

## Purpose

Declares the surface implemented in [`Image.cpp`](Image.cpp.md) (the raster container and the uncompressed-image writer) and [`ImageJPEG.cpp`](ImageJPEG.cpp.md) (the lossy codec bridge). This is a *screenshot and loading-screen* facility, not the texture path — game textures are block-compressed and go straight to the graphics device without ever becoming one of these.

## Exported units

- **`ImageDataFormat`** — the pixel layout: unknown, three bytes per pixel, or four. There is no separate channel-order axis; the order is fixed by the format.
- **`Image`** — the raster: width, height, format, channel count, a pixel buffer and a flag recording whether this object owns that buffer. Constructible around a buffer the caller already has (borrowed) or filled by decoding (owned).
- **`Image.OpenJPEG`** — decode from a reader or from a byte range into this image, taking ownership of the result.
- **`Image.SaveTGA`** — write the uncompressed container, to an engine writer or straight to a host file, in this image's format or in a requested one, with or without row padding.
- **`Image.SaveJPEG`** — encode to an engine writer at a given quality, optionally with the rows in bottom-up order.
