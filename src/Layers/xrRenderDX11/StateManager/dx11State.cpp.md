# src/Layers/xrRenderDX11/StateManager/dx11State.cpp

> Compiles a material pass's recorded state block into shared, deduplicated device state objects — once, at load time — and binds that set in one call at draw time.

**Needs** — [`dx11State.h`](dx11State.h.md) · [`dx11StateCache.h`](dx11StateCache.h.md) · [`dx11SamplerStateCache.h`](dx11SamplerStateCache.h.md) · [`../dx11StateUtils.h`](../dx11StateUtils.h.md) · [`xrRender/Blender_Recorder.h`](../../xrRender/Blender_Recorder.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11State.h`](dx11State.h.md)
**Tier floor** — T1: it holds non-owning references to device objects whose lifetime is the process, and binds them from the innermost draw loop.

## Purpose

Chapter 18 loads a material as a list of passes, and each pass records its render state as a little program — a sequence of "set this field to that value" operations captured from the material's text description. That recorded form is flexible and slow. This file is where it is **resolved once into device objects**, and it is the heart of what "the state manager" means in this backend.

The resolution has two halves. The recorded program is replayed into four descriptions — rasterizer, depth-stencil, blend, and one sampler description per texture slot per stage — and each description is then *interned*: handed to a cache that returns an existing device object when an identical description has been seen, and creates one otherwise. A level's several thousand material passes collapse into a handful of distinct rasterizer states, a few dozen depth-stencil states and a similar number of blend and sampler states.

## State

```text
RECORD PassState
  rasterizer    : handle        # non-owning: the cache owns it and it lives for the process
  depth_stencil : handle        # non-owning
  blend         : handle        # non-owning
  samplers      : per-stage list<SamplerHandle>   # index = slot; a gap holds "none"
  stencil_ref   : int           # "unset" is a distinct value, see below
  alpha_ref     : int           # 0..255; emulated, not device state
```

Invariants: the three device handles are *weak*. Nothing here releases them, and nothing may, because the caches hold the only ownership and clear everything at device teardown. The consequence a rebuild must respect: **a pass state must never outlive a device reset**, and the reset path therefore rebuilds every material.

The sampler list per stage is truncated to the highest slot the pass actually uses, so a pass that uses two textures does not carry an array of sixteen handles.

## `Create`

**Contract** — builds a pass state from a recorded state block. Replays the block to produce the three pipeline descriptions and the per-stage sampler descriptions, interns each, and returns the assembled record. Allocates. Called at material load time, never per frame.

```text
FUNCTION create(recorded_block) -> PassState
  s = new PassState
  recorded_block.replay_into(s)            # fills stencil_ref / alpha_ref
  s.rasterizer    = rasterizer_cache.intern(recorded_block)
  s.depth_stencil = depth_cache.intern(recorded_block)
  s.blend         = blend_cache.intern(recorded_block)
  FOR EACH stage IN {vertex, pixel, geometry, hull, domain, compute}
    s.samplers[stage] = intern_samplers(recorded_block, base_slot_of(stage))
  RETURN s
```

**Notes** — The source flags that interning is not locked, and that it would need to be if materials were loaded from more than one thread. As written, material loading is single-threaded; a rebuild that parallelizes it must guard the caches.

## `intern_samplers`

**Contract** — extracts the sampler settings for one stage from the recorded block and turns them into a compact handle list.

```text
FUNCTION intern_samplers(block, stage_base_slot) -> list<SamplerHandle>
  descs = slot-indexed array, every entry reset to the default sampler
  used  = slot-indexed array of false
  block.fill_sampler_descriptions(descs, used, stage_base_slot)
  highest = highest slot with used = true
  IF none THEN RETURN empty
  FOR slot IN 0 .. highest
    result[slot] = used[slot] ? sampler_cache.intern(descs[slot]) : none
```

**Invariants** — The recorded block addresses samplers in the *flat* slot space (stage base plus the program's own bind point) that the constant parser produced; passing the stage's base is what selects this stage's window of it. Gaps below the highest used slot are preserved as explicit "none" entries rather than being compacted, because slot positions are what the shader binds by.

## `Apply`

**Contract** — pushes this pass's whole state onto a command list: the three pipeline objects, the stencil reference if the pass declared one, the alpha reference, and the six per-stage sampler arrays. Does not touch the device directly — it goes through the command list's state manager, which filters redundancy and defers.

**Notes** — `stencil_ref` uses an out-of-range sentinel to mean "this pass does not care", in which case the previous reference value is left alone. That matters because the stencil reference is not part of the depth-stencil object here: two passes that differ only in reference value share one object, and the reference is set alongside it.

The alpha reference is **not device state on this backend**. It is pushed to the state manager, which writes it into a named shader constant. See [`dx11StateManager.cpp`](dx11StateManager.cpp.md).

## `Release`

**Contract** — destroys the pass state record. Releases nothing device-side, by design, because everything it points at is cache-owned.
