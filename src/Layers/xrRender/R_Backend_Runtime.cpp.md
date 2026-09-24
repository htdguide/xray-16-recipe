# src/Layers/xrRender/R_Backend_Runtime.cpp

> The command list's lifecycle and its two widest operations: dropping every cached assumption, and installing a whole pass's texture list across every shader stage.

**Needs** — [`R_Backend.h`](R_Backend.h.md) · [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [`LightTrack.h`](LightTrack.h.md) · [`R_Backend_hemi.h`](R_Backend_hemi.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md)
**Tier floor** — T1: it manipulates raw device handles, per-stage binding tables and sentinel bit patterns, and its cost model is the reason the surrounding class exists.

## Purpose

The command list ([`R_Backend.h`](R_Backend.h.md)) is a shadow copy of the graphics device's state that exists to skip redundant device calls. Three operations in that design cannot be written as a one-line comparison and live here: **invalidation** (what "I no longer know what the device holds" means, field by field), **texture-list installation** (the only setter that touches every shader stage and that has to clean up after the previous pass), and the **frame bracket** that puts the two at the right moments. The per-object lighting publication sits here too, because it is the one place where a bound texture's *description data* feeds back into a shader constant.

Everything in this file is the part of the command list that is too long to inline. The inline half is in [`R_Backend_Runtime.h`](R_Backend_Runtime.h.md); the shape of the state both halves operate on is in [`R_Backend.h`](R_Backend.h.md).

## State

Stateless in its own right — every field it touches belongs to the command list.

## `Invalidate`

**Contract** — Declares every cached field unknown. Clears the bound targets, the depth target, the vertex layout, both geometry buffers and the stride; clears the state block and every shader-stage program; clears the pointers to the current pass's texture, matrix and constant lists; writes the **impossible sentinel** into all fourteen fixed-state fields; unmaps the transform cache's bindings; clears every per-stage texture slot and every fixed-function matrix slot; and returns the list to the immediate submission context. Touches the device only where the device offers a "forget your own state" call. Allocates nothing, never fails.

**Invariants** — The distinction between *clearing to nothing* and *setting the sentinel* is the whole point and is not arbitrary:

- Fields that hold a **handle or a pointer** are cleared to nothing. Nothing is already an impossible value for them — no real program, buffer or texture is nothing — so the next set compares unequal and reaches the device.
- Fields that hold a **small integer** (the eight stencil fields, the cull and fill modes, the depth enable and comparison, the alpha reference, the colour write mask) are set to the all-bits-one sentinel, because every plausible value for them, *including zero*, is also a legal value the device might currently hold. Initialising these to zero is the classic way to get this wrong: the first draw of the frame then asks for "cull off", the shadow already says "cull off", the device call is skipped, and the device is still culling from whatever the last frame left.

**Notes** — The sentinel is not an enum value to be added to the state vocabulary. It is "a bit pattern no legal value can equal", and a rebuild in a language with option types should say `none` and mean the same thing.

The transform bindings must be unmapped here for a reason that is one level deeper than the rest: on a device where constants live in buffers that are themselves unmapped at this moment, a binding is an offset into a buffer that no longer exists. The same is true of every other constant sub-cache, which is why they are unmapped together.

## `OnFrameBegin` / `OnFrameEnd`

**Contract** — `OnFrameBegin` invalidates, restores the renderer's normal-depth mode, binds the base colour and depth targets, zeroes the statistics and turns stencil off. `OnFrameEnd` asks the device to reset its own state where it can, then invalidates. Both do nothing at all when the process is running as a dedicated server, which has no graphics device.

**Invariants** — Invalidating at *both* ends of the frame is not redundancy. The end-of-frame invalidation pairs with the device's own state reset, so the shadow and the device agree on "nothing is bound". The start-of-frame invalidation exists because anything at all may have run between the two — a screenshot, a device reset, a debug overlay drawing through its own path — and the command list must not assume it did not.

**Notes** — The statistics are zeroed at frame *begin*, not frame end, so a reader sampling them at any point sees this frame's totals so far rather than a mixture of two frames.

## `set_Textures`

**Contract** — Installs a whole pass's texture list. Each entry is a (flat stage number, texture) pair; the number is decoded into (shader stage, slot within that stage), compared against the shadow for that slot, and bound through the texture's own bind hook when it differs. After the list is consumed, **every slot above the highest one this list touched is cleared, per stage**. No early-out on "same list object as last time". Does nothing on a dedicated server.

```text
FUNCTION set_textures(list)
  current_list = list
  highest_touched = map from stage to -1        # -1, not 0: a list that touches
                                                # nothing must clear from slot 0

  FOR EACH (flat_number, texture) IN list
    (stage, slot) = decode_stage(flat_number)   # partition described in R_Backend.h
    highest_touched[stage] = max(highest_touched[stage], slot)

    IF bound[stage][slot] != texture
       OR (texture is present AND texture.bound_slice != texture.wanted_slice)
      bound[stage][slot] = texture
      count_a_texture_change()
      IF texture is present
        texture.bind(this, flat_number)         # the texture decides what a bind means:
                                                # an ordinary surface, a video frame,
                                                # a sequence frame, or a load placeholder
        texture.bound_slice = texture.wanted_slice

  FOR EACH stage
    FOR slot FROM highest_touched[stage] + 1 TO stage.slot_count - 1
      IF bound[stage][slot] IS present
        bound[stage][slot] = none
        unbind at (stage, slot)
```

**Invariants**

- **The trailing clear is mandatory.** Leaving a texture bound in a slot the current program does not sample is harmless on one device and a validation failure or a read/write hazard on another — the classic case being a render target still bound as a source while the same resource is being written. A rebuild that skips the clear will work until it does not.
- The comparison is **per slot**, never per list. Two passes may share one texture list object and still need different views of it, which is why the obvious "same list, nothing to do" shortcut is deliberately disabled in the source. The second half of the comparison — the slice selector — is how an array texture addresses a different layer without becoming a different texture object.
- The starting value of "highest touched" is one below the first slot, so a stage the list mentions not at all is cleared from its very first slot. Starting at zero would leave slot 0 of every unmentioned stage holding a stale texture.

**Notes** — Only the vertex and pixel stages exist on every backend; geometry, hull, domain and compute stages exist only where the device has them, and their blocks in the flat number space simply go unused otherwise. The clear loops skip slots already known empty, which matters because the clear is per-slot on this device generation and the loops run for every pass.

## `apply_lmaterial`

**Contract** — Publishes the per-object lighting values the caller latched on the command list — the hemispherical ambient term, the sun visibility term and the six-face ambient cube — together with the **material coordinate** derived from the texture bound at the current pass's base sampler. Returns immediately, doing nothing, if the current pass declares no base sampler. Reads the ambient cube in a fixed face order.

```text
FUNCTION apply_lmaterial()
  location = lookup_constant("s_base")      # FROZEN name of the base-colour sampler
  IF location IS none THEN RETURN
  # the name must resolve to a sampler, not a numeric constant

  texture  = texture bound at location.sampler_slot
  material = texture.material               # small integer from the texture description

  hemi.set_material(object_ambient, object_sun, 0, (material + 0.5) / 4)
  hemi.set_pos_faces(cube[POS_X], cube[POS_Y], cube[POS_Z])
  hemi.set_neg_faces(cube[NEG_X], cube[NEG_Y], cube[NEG_Z])
```

**Invariants**

- `s_base` is a frozen name: it is how a pass declares "this is my base colour texture", and the material a surface is made of is a property of that texture, not of the mesh. A rebuild must keep the name and must keep the rule that the material comes from the base texture.
- `(material + 0.5) / 4` is a texture coordinate, not a scale. The material index selects one of **four** rows in a lookup texture holding the shipped lighting response curves; the half-step lands the sample in the middle of a row instead of on the boundary between two, where filtering would blend two materials. The divisor is tied to "four material rows" and moves only with it.
- The ambient cube's face order — positive X, Y, Z in one constant, negative X, Y, Z in the other — must match the order the light-tracking pass fills. Getting it wrong produces a lit, plausible, wrongly-directional image.

**Notes** — A debug console flag can override the material for every surface at once, which is how the shipped lighting response table was tuned. It is compiled out of a shipping build and a rebuild may drop it.

## `SetupStates`

**Contract** — Applies the state that is global rather than per-pass: the default winding order (counter-clockwise is front-facing, which is the convention the shipped meshes are authored in), the anisotropic filtering limit, and the mip level-of-detail bias. The latter two come from console variables, so they are player settings rather than engine constants.

## `OnDeviceCreate` / `OnDeviceDestroy`

**Contract** — `OnDeviceCreate` acquires the device's debug-annotation channel if it has one, builds the two debug-draw geometry descriptions (see [`R_Backend_DBG.cpp`](R_Backend_DBG.cpp.md)), and invalidates. `OnDeviceDestroy` tears down the same three in reverse.

## `set_ClipPlanes`

**Contract** — Two forms: a list of planes, and a transform plus a face mask from which the planes are derived as a frustum. **Both are inert on every backend this engine still ships against.** The transform form still builds the frustum before handing it on, so the cost is paid and discarded.

**Notes** — User clip planes were a fixed-function feature. On a programmable pipeline the equivalent is an output the vertex program writes, which means the clipping would have to be expressed in each of the shipped shader sources — and those are frozen. So the feature is gone and its call sites remain. A rebuild should delete the entry points rather than reproduce dead ones, and note that nothing in the shipped data needs them.
