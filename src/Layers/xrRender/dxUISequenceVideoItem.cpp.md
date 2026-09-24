# src/Layers/xrRender/dxUISequenceVideoItem.cpp

> Grabs whichever texture the currently bound material uses as its base, so a scripted video sequence can drive that texture's playback.

**Needs** — [`Include/xrRender/UISequenceVideoItem.h`](../../Include/xrRender/UISequenceVideoItem.h.md) · [`dxUISequenceVideoItem.h`](dxUISequenceVideoItem.h.md) · [`r_constants.h`](r_constants.h.md) · [`R_Backend.h`](R_Backend.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`dxUISequenceVideoItem.h`](dxUISequenceVideoItem.h.md)
**Tier floor** — T1: it reads a sampler's binding slot out of the bound material's reflected constant table and then reads the texture the backend has bound there.

## Purpose

Cut scenes and menu backgrounds are authored as a *sequence* of items, some of which are videos. The engine half owns the sequence, the item timing and the script callbacks. The renderer half owns exactly one thing: a handle to the texture whose decoder is playing, so the engine can start, stop, seek and query it without knowing what a texture is.

The interesting decision is how that handle is obtained. There is no name lookup and no registry: the item is captured *implicitly*, from whatever the renderer currently has bound, at the moment the sequence tells it to.

## State

```text
RECORD UIVideoItem
  texture : optional<Texture>   # borrowed, never owned
```

Invariant: the texture is borrowed. It belongs to the resource manager and to the material that named it; this object only points at it, and releasing it here would free a resource other materials share. Clearing the field is the only "release" that exists.

## `capture_texture`

**Contract** — Adopt the texture currently bound to the base-colour sampler slot. Requires that a material has been bound and a batch is in flight; the result must be non-empty, and an empty one is fatal, because a video item with no texture can never play and the failure would otherwise surface much later as silence.

```text
FUNCTION capture_texture()
  # Which slot is "the base texture" is not fixed: it is whatever slot the
  # bound material's reflected constant table gave the conventional base-colour
  # sampler name. Slot zero is the fallback for a material that does not
  # declare one at all.
  binding = bound_material.constants.lookup(base_colour_sampler_name)
  slot    = binding is present ? binding.sampler_index : 0
  texture = currently_bound_texture(slot)
  FAIL WITH "no texture bound" IF texture is none
```

**Notes** — The base-colour sampler's conventional name is fixed by the shipped shader sources and must be reproduced exactly; see [`r_constants.cpp`](r_constants.cpp.md) for how the constant table is built and searched. The whole indirection exists because the same UI material family is authored with different sampler layouts across the three games, and the engine may not hard-code a slot.

## `has_texture` / `reset_texture`

**Contract** — Report whether a texture was captured; drop the reference without releasing the resource. `reset_texture` is what ends an item — the next one captures afresh.

## `is_playing` / `synchronise` / `play` / `stop`

**Contract** — Pure delegations to the captured texture's video decoder. `play` takes a loop flag and a start time, with a sentinel meaning "from the beginning". `synchronise` advances the decoder to a given time, which is how the sequence keeps video, sound and script callbacks on one clock. All four require a captured texture; calling them without one is a caller error.

**Notes** — That these are delegations rather than logic is the point of the interface: the engine must be able to drive video playback without linking the video decoder, and every decision about decoding, frame timing and upload lives in the texture. Which part of the engine steps the clock is the engine's business; this file only forwards.

## `copy`

**Contract** — Adopt another item's contents, including its borrowed texture reference. Used when the engine duplicates a sequence.
