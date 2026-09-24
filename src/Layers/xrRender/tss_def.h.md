# src/Layers/xrRender/tss_def.h

> Declares the recorded state list a material pass builds up, and the conversions from it into a backend's state objects.

**Needs** — [`tss_def.cpp`](tss_def.cpp.md) · [`tss.h`](tss.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`ResourceManager.h`](ResourceManager.h.md) · [`SH_Atomic.h`](SH_Atomic.h.md) · [`tss.h`](tss.h.md) · [`tss_def.cpp`](tss_def.cpp.md)
**Tier floor** — T2: a declaration plus one small record.

## Purpose

Declares the type implemented in [`tss_def.cpp`](tss_def.cpp.md): the list of render-state assignments a material pass accumulates while it is being recorded, and the translation of that list into whatever form the device actually wants.

## State

```text
RECORD StateAssignment
  category : ENUM { render_state, texture_stage_state, sampler_state }
  a, b, c  : int    # meaning depends on category:
                    #   render_state:        a = which state,  b = value
                    #   texture_stage_state: a = stage, b = which state, c = value
                    #   sampler_state:       a = slot,  b = which state, c = value

RECORD StateList
  assignments : list<StateAssignment>
```

Invariants:

- The three categories share one record shape with positional fields. That is a space decision — a pass records a few dozen assignments and they are compared wholesale — and it is why equality can be a raw memory comparison.
- At most one assignment exists per addressable target: setting a state that is already in the list replaces it rather than appending. The list is therefore a *set of final values*, not a journal, and its order is the order of first assignment.

## Exported units

- **`StateList`** (`SimulatorStates`) — `set_render_state`, `set_texture_stage_state`, `set_sampler_state`, `equal`, `clear`, `record`, and the four conversions that fill a backend's rasterizer, depth-stencil, blend and sampler descriptions, plus one that extracts the two values that are not part of any description. All described in [`tss_def.cpp`](tss_def.cpp.md).
