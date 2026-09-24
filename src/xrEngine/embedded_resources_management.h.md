# src/xrEngine/embedded_resources_management.h

> Gets the splash image and the window icon into the process — from the executable's own resource table on Windows, from files beside the game on everything else.

**Needs** — [`xr_3da/resource.h`](../xr_3da/resource.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`Device_Initialize.cpp`](Device_Initialize.cpp.md) · [`x_ray.cpp`](x_ray.cpp.md)
**Tier floor** — T1 on the Windows path: it reads a bitmap's raw pixel block, its stride and its bit depth, and wraps that memory as a surface without copying. T3 on the other path, which is a file load.

## Purpose

Two images are needed before the graphics device exists: the splash shown while the level
loads, and the window icon, which differs per game (*Shadow of Chernobyl*, *Clear Sky*,
*Call of Pripyat* each have their own). Neither can come from the virtual filesystem,
because on the splash's path the filesystem is what is still being mounted.

Windows lets an executable carry them; other platforms do not, so there the same images are
plain files expected beside the configuration file that defines the filesystem roots.

## State

Stateless.

## `ExtractSplashScreen`

**Contract** — returns the splash image as a surface the windowing layer can blit, or
nothing. On Windows it is decoded from the executable's resource table by numeric id; on
other platforms it is read from `logo.bmp` in the working directory. Allocates. Failure is
reported by returning nothing; the splash is optional.

## `ExtractAndSetWindowIcon`

**Contract** — sets a window's icon from a numeric image id. On Windows the icon is pulled
from the resource table and installed through the platform's window message, at both the
small and large sizes, because the platform keeps two and picks per context. Elsewhere the
id selects among three files named per game and the icon is handed to the windowing layer
directly. Silently does nothing when the image is missing.

**Notes** — reaching the native window handle to send that message is the one place the
splash path goes underneath the windowing seam. A rebuild whose windowing layer has an
icon-setting call uses it and this asymmetry vanishes.

## The vertical flip

**Contract** — flips a surface's rows in place, without allocating a row buffer, by swapping
bytes pairwise between the top and bottom halves.

**Notes** — this exists because the two worlds disagree about which row comes first: the
platform's bitmap resources are bottom-up, the surface type is top-down. The in-place,
byte-at-a-time swap is chosen over a row buffer because the image is loaded once at startup
and an allocation at that point is avoidable. That is a nearly irrelevant saving and a
rebuild should not imitate it; what must survive is *the flip itself*, because getting it
wrong produces an upside-down splash and nothing else complains.

## Notes

The resource ids are shared with the executable's build description — the same header names
them. In a rebuild where images are ordinary data files in the game's filesystem, this
whole file disappears and only the ordering constraint remains: the splash must be
displayable before the virtual filesystem is mounted.
