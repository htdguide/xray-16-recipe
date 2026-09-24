# src/Layers/xrRenderDX11/StateManager/dx11ShaderResourceStateCache.cpp

> Shadows which readable resource sits in which slot of which stage, and turns a frame's scattered texture changes into one contiguous rebind per stage per draw.

**Needs** — [`dx11ShaderResourceStateCache.h`](dx11ShaderResourceStateCache.h.md) · [`../dx11HW.h`](../dx11HW.h.md) · [`../dx11SH_Texture.cpp`](../dx11SH_Texture.cpp.md)
**Used by** — [`dx11ShaderResourceStateCache.h`](dx11ShaderResourceStateCache.h.md)
**Tier floor** — T1: a fixed-size shadow array per stage with a dirty range, touched several times per draw.

## Purpose

Textures change more often than anything else in a frame: every material change rebinds two to five of them. Binding each one individually costs a device call each; binding all slots of a stage costs one call but re-sends up to sixteen entries. This cache takes the third option — **shadow every slot, remember the lowest and highest slot that changed, and bind exactly that run once per draw**.

It is the direct counterpart of the constant-table diff in [`../dx11R_Backend_Runtime.h`](../dx11R_Backend_Runtime.h.md), applied to a different resource kind, and it is the single most transferable idea in this directory for a rebuild on any API where binding is per-slot.

## State

```text
RECORD ResourceSlotCache                    # one per command list
  views      : per-stage list<handle>       # fixed slot count per stage, differs per stage
  dirty_low  : per-stage int                # lowest slot changed since the last bind
  dirty_high : per-stage int                # highest
  dirty      : per-stage bool
```

Invariant: when `dirty` is false for a stage, the device's binding for that stage equals `views[stage]`. The range is *inclusive on both ends*, and it covers unchanged slots in the middle — that is the trade being made, and it is the right one because slots used by one material are adjacent.

## `SetPSResource` and its five siblings

**Contract** — record a resource in one slot of one stage. Does nothing when the slot already holds it. Otherwise stores it and widens that stage's dirty range (or opens a new one). Asserts the slot is within the stage's capacity.

```text
FUNCTION set(stage, slot, view)
  IF views[stage][slot] == view THEN RETURN
  views[stage][slot] = view
  IF dirty[stage] THEN
    dirty_low[stage]  = min(dirty_low[stage], slot)
    dirty_high[stage] = max(dirty_high[stage], slot)
  ELSE
    dirty[stage] = true ; dirty_low[stage] = dirty_high[stage] = slot
```

## `Apply`

**Contract** — for each stage with a dirty range, bind the slots from low to high inclusive in one call, then clear that stage's range. Called at the top of every draw and every dispatch, before anything else, because unbinding a render target so it can be read (see the target path) only takes effect once the read binding is issued.

## `ResetDeviceState`

**Contract** — clears every shadow and every dirty range. Required whenever the device's actual bindings are changed behind the cache's back — after a device reset, and at the start of a frame that begins by binding nothing.

**Notes** — This function does not clear the compute stage's shadow array, its range, or its dirty flag, while every other stage is cleared. The omission looks like an oversight rather than a decision: nothing else treats the compute stage differently, and a stale compute shadow makes the first compute binding after a reset a silent no-op. A rebuild should clear all six.
