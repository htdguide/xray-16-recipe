# src/xrCore/PostProcess/PostProcess.hpp

> Declares the animated screen-effect: eleven parameter tracks, each a keyframed curve, driven by one playback clock.

**Needs** — [`PostProcess.cpp`](PostProcess.cpp.md) · [`PPInfo.hpp`](PPInfo.hpp.md) · [`Animation/Envelope.hpp`](../Animation/Envelope.hpp.md) · [`FS.h`](../FS.h.md)
**Used by** — [`PostProcess.cpp`](PostProcess.cpp.md) · [`PostprocessAnimator.cpp`](../../xrGame/PostprocessAnimator.cpp.md) · [`PostprocessAnimator.h`](../../xrGame/PostprocessAnimator.h.md)
**Tier floor** — T1: the file format it loads is a versioned binary stream whose field order is frozen.

## Purpose

Declares the surface implemented in [`PostProcess.cpp`](PostProcess.cpp.md). An authored screen effect — a grenade flash, a psi attack, a drug taking hold — is a file of eleven keyframed curves, one per parameter of [`PPInfo.hpp`](PPInfo.hpp.md). This header fixes the parameter *order*, which is the file format's field order and therefore frozen.

## Exported units

- **`pp_params`** — the eleven parameters, in file order and by index: base colour (0), additive colour (1), grey colour (2), grey amount (3), blur (4), horizontal duality (5), vertical duality (6), noise intensity (7), noise grain (8), noise rate (9), colour-map influence (10). The count and the end marker are both derived from this list. **This order is the file layout** — see [`PostProcess.cpp`](PostProcess.cpp.md).
- **`POSTPROCESS_FILE_VERSION`** — the current version tag, 2. Version 1 files exist and are still read; what the tag gates is described in the implementation.
- **`POSTPROCESS_FILE_EXTENSION`** — the extension that identifies one of these files; a file with any other extension is rejected outright.
- **`CPostProcessParam`** — the interface one animated parameter must satisfy: advance to a time, load, save, report total length and key count, report one key's time, and insert, delete, update, read or clear keys. The editing half of that surface exists for the effect editor, not for the game.
- **`CPostProcessValue`** — a single-channel parameter: one keyframed curve writing into one float of the parameter block.
- **`CPostProcessColor`** — a three-channel parameter: three keyframed curves writing into one colour of the parameter block, plus a stored base scalar that is serialized but never applied.
- **`BasicPostProcessAnimator`** — one loaded effect: the eleven parameters, the accumulated parameter block they write into, a name, a current and desired intensity factor with a rate between them, a cyclic flag and the total length. Loads, evaluates at a time, and hands out the resulting block.
