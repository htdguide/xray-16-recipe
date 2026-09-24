# src/Layers/xrRenderDX11/dx11r_screenshot.cpp

> Reading the finished frame back off the device, for the three purposes the engine has: a picture for the player, a thumbnail inside a save file, and a tile of a level or environment map for the tools.

**Needs** — [`dx11HW.h`](dx11HW.h.md) · [`xrCore/Media/Image.hpp`](../../xrCore/Media/Image.hpp.md) · [`xrEngine/xrImage_Resampler.h`](../../xrEngine/xrImage_Resampler.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r4_rendertarget_build_textures.cpp`](../xrRenderPC_R4/r4_rendertarget_build_textures.cpp.md)
**Tier floor** — T1: it maps device memory and rewrites pixels in place.

## Purpose

One entry point, three modes, and the reason they are one function is that they share the expensive part: **getting the presented image from device memory into host memory**, which stalls the pipeline and is therefore never done casually.

## `Screenshot`

**Contract** — reads back the base render target and writes it somewhere, according to the mode. Blocks until the device has finished the frame. Reports and returns without writing anything if the target cannot be obtained or captured — a screenshot is never worth killing the process for.

```text
FUNCTION screenshot(mode, name)
  source = the base render target's underlying allocation
  IF absent THEN report; RETURN
  image = capture(source)          # allocate host-readable storage, copy, map, read
  IF failed THEN report; RETURN

  CASE mode OF
    for a save file:
      resize to 128 x 128 with a box filter
      compress to the four-bit-per-texel block format
      write as an image container in the legacy layout, under the given name
    for the player:
      name it from the user name, a timestamp and the current level (or the main menu)
      write as a lossy compressed image into the screenshots directory
      IF the high-quality switch is on THEN also write a lossless copy
    for a level map or an environment map tile:
      copy into the pre-allocated host-readable staging target and map it
      swap the red and blue channels in place, preserving alpha
      resample to a *square* of the frame's height with a high-quality filter
      write losslessly under the caller's name
```

**Invariants** — The save-file thumbnail is **128 pixels square, block-compressed, in the legacy container layout**. All three are frozen: the save format embeds this image and the original engine reads it back.

The level-map and environment-map modes resample to a square whose edge is the frame's *height*, which is how a wide frame becomes the square tile those tools assemble. The engine is expected to have been told to render a square-aspect view beforehand; this step only crops the buffer to square.

**Notes** — The channel swap in the tool modes is the visible consequence of the format-order mismatch noted in [`dx11TextureUtils.cpp`](dx11TextureUtils.cpp.md): the device's four-channel layout and the image writer's disagree, and the cheapest fix at this volume is an in-place rewrite of each pixel.

The tool modes copy into a **pre-allocated** host-readable target owned by the render-target set, rather than allocating one per call, because they are called once per tile over hundreds of tiles.

The player-facing path writes a lossy image by default and a lossless one only on a command-line switch: a full-resolution lossless frame is tens of megabytes and the switch exists for people capturing reference images.
