# src/Layers/xrRenderDX11/StateManager/dx11StateManager.cpp

> The per-command-list state front end: takes both whole resolved state blocks and individual legacy-style field changes, reconciles them lazily, and binds at most three objects per draw.

**Needs** — [`dx11StateManager.h`](dx11StateManager.h.md) · [`dx11StateCache.h`](dx11StateCache.h.md) · [`../dx11StateUtils.h`](../dx11StateUtils.h.md) · [`../dx11HW.h`](../dx11HW.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11StateManager.h`](dx11StateManager.h.md)
**Tier floor** — T1: it is consulted on every draw and reads descriptions back out of driver objects.

## Purpose

Two kinds of caller set render state and they want opposite things. A **material pass** hands over a fully resolved state block — three device objects, ready to bind. A large amount of older engine code instead sets *one field at a time*: "depth test off", "cull the other way", "write only alpha" — the vocabulary of the fixed-function generation, which the shared renderer code in chapters 18 and 19 still speaks.

This file makes both work without either paying for the other. The resolved path is a pointer comparison and a bind. The field path reconstructs a description, mutates one field, and re-interns at draw time — expensive, but only when it is used, and its cost is a cache lookup rather than an object creation once the resulting combination has been seen.

## State

```text
RECORD StateManager                 # one per command list
  bound        : {rasterizer, depth_stencil, blend} handles   # non-owning
  stencil_ref  : int
  alpha_ref    : int                                          # 0..255
  alpha_ref_constant : Constant     # the named shader constant alpha_ref is written into
  sample_mask  : int (32-bit)

  # three parallel three-flag state machines, one per pipeline object kind:
  need_apply   : bool   # the bound object is not what the device has
  changed      : bool   # the description was edited; re-intern before binding
  desc_stale   : bool   # the description does not describe the bound object; refresh it

  desc         : {rasterizer, depth_stencil, blend} descriptions
  override_scissor : optional<bool>   # forces the scissor bit regardless of what passes set
```

Invariants per kind: `changed` implies the description is authoritative and the handle is not; `desc_stale` implies the handle is authoritative and the description is not. They are never both true — setting a resolved object clears `changed` and sets `desc_stale`; editing a field first refreshes the description (clearing `desc_stale`) and then sets `changed`.

## `Apply`

**Contract** — reconciles and binds. For each of the three kinds: if the description was edited, intern it to get the object; if the object differs from what the device holds, bind it. Called from the draw path, after resource binding and before the constant flush.

```text
FUNCTION apply()
  FOR EACH kind IN {rasterizer, depth_stencil, blend}
    IF NOT (need_apply[kind] OR changed[kind]) THEN CONTINUE
    IF changed[kind] THEN
      bound[kind] = cache_for(kind).intern(desc[kind])
      changed[kind] = false
    device.bind(kind, bound[kind], extra)
    need_apply[kind] = false
  # extra: the depth-stencil bind carries the stencil reference,
  #        the blend bind carries the blend factor and the sample mask
```

**Notes** — The stencil reference and the sample mask are *not* part of their objects, so changing either sets `need_apply` without setting `changed`: the same object is rebound with a different parameter, and no new object is created. Recognizing which parameters are object-external is what keeps the caches small; a rebuild on an API where they are baked into a pipeline object will see a combinatorial increase and should plan for it.

The blend factor is a fixed zero vector — the engine never uses a constant blend colour. It is passed explicitly because the binding call demands it.

## The field-level setters

**Contract** — `SetStencil` (the whole eight-parameter stencil configuration at once), `SetDepthFunc`, `SetDepthEnable`, `SetColorWriteEnable`, `SetCullMode`, `SetFillMode`, `SetMultisample`, `SetSampleMask`, `EnableScissoring`. Each refreshes its description if stale, translates the caller's legacy enumeration into the device's, compares, and marks changed only on a real difference.

```text
FUNCTION refresh_description(kind)
  IF NOT desc_stale[kind] THEN RETURN
  IF bound[kind] EXISTS THEN desc[kind] = describe(bound[kind])   # read back from the object
  ELSE                       desc[kind] = defaults
  desc_stale[kind] = false
```

**Invariants** — Reading the description *back out of the bound object* rather than keeping a shadow copy is what makes the two paths compose: a material pass binds an object this manager never built a description for, and the next field-level edit still starts from exactly that object's settings.

Front and back stencil faces are always set identically — the engine has no two-sided stencil technique, so the eight-parameter call writes both faces from one set of values. Colour write masks are likewise applied to the first four render targets uniformly.

A stencil configuration whose enable flag is false stops after the flag: the remaining parameters are not recorded, so two passes that disable stencil with different leftover parameters share one object.

## Alpha reference

**Contract** — `SetAlphaRef` records a 0..255 cut-off and, if a constant is currently bound for it, writes it as a normalized float (value divided by 255) into that shader constant. `BindAlphaRef` supplies the constant — the material's shader declares it by name — and pushes the current value immediately. `UnmapConstants` drops the binding, and must be called whenever the constant table changes, because the constant record belongs to that table.

**Notes** — This is the concrete reason the draw path flushes constants *after* applying state: applying a pass state sets the alpha reference, which writes a constant. On the fixed-function generation this was a device state; here alpha cut-out is done in the shader, so the state manager's job is to keep the old call site working by routing the value to a constant. A rebuild that drops the legacy call sites can delete this; a rebuild that keeps them must keep the ordering.

## `OverrideScissoring`

**Contract** — forces the scissor enable bit on or off regardless of what subsequent pass states specify, and restores the pass's own value when the override is lifted. Used by the interface layer, which clips whole widget trees with a scissor rectangle and cannot control what state the widgets' materials set.

**Notes** — While the override is active, every resolved-state application re-applies the forced bit, which means those applications go down the description-editing path and re-intern. That is the cost of the feature, and it is bounded because the override is only on during interface drawing.

## `Reset`

**Contract** — returns the manager to a known state: no objects bound, all descriptions defaulted, everything marked as needing application, no constant binding, full sample mask, no scissor override. Needed after the device's state is changed behind the manager's back.
