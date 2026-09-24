# src/Layers/xrRender/blenders/blender_deffer_model.cpp

> The material template every animated model in the game is drawn with: it decides, per model, whether that model goes down the deferred g-buffer path or is detoured to a forward-lit pass, and it emits the shadow-map variant.

**Needs** — [`blender_deffer_model.h`](blender_deffer_model.h.md) · [`uber_deffer.h`](uber_deffer.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`r2_types.h`](../../xrRender_R2/r2_types.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`blender_deffer_model.h`](blender_deffer_model.h.md)
**Tier floor** — T2: it is a decision tree over booleans that emits names. What stops T3 is the parameter block, which is read from and written to the shipped material library as a byte image.

## Purpose

This is the template behind the class identifier `"MODEL   "` — the one every skinned or rigid dynamic object in the shipped data names. Its whole job is to answer one question per material: *can this surface be written into the g-buffer at all?* A g-buffer stores one opaque sample per pixel, so a surface that is genuinely translucent, or that the artist marked as needing painter's-order sorting, cannot live there and must be drawn later, lit only approximately, in a forward pass.

Everything else it does is delegation: the deferred case hands off to [`uber_deffer`](uber_deffer.cpp.md), which assembles the actual shader names from the texture set; the shadow case emits a depth-only pass with or without alpha testing.

## State

```text
RECORD DefferModelParams        # the parameter block, as stored in the material library
  use_alpha_channel : bool, default false   # "the base texture's alpha means something"
  alpha_ref         : int in [0,255], default 32
  tessellation      : enum { NO_TESS=0, TESS_PN=1, TESS_HM=2, TESS_PN_HM=3 }, default NO_TESS
```

**Invariants**

- The template's parameter-block version is **2**, and the save/load pair must honour all three historical versions, because materials authored against older engine builds are still in the shipped library:
  - version 0 — no parameters stored at all; the loader supplies the defaults above and reads nothing.
  - version 1 — the alpha flag then the alpha reference, in that order.
  - version 2 and later — those two, then the tessellation token.
  A loader that reads the version-2 layout from a version-0 record will consume bytes belonging to the next material and corrupt the rest of the library.
- The tessellation field is a *token* property: the writer emits the selection plus the four `(id, label)` pairs `NO_TESS`, `TESS_PN`, `TESS_HM`, `TESS_PN+HM` so that an authoring tool can present a menu without knowing the enum. The labels are part of the written record; the reader consumes them.

## `compile(context)`

**Contract** — emits the pass list for one element of one model material. Reads the parameter block and the renderer's capability flags; writes passes into the recorder. No allocation beyond the recorder's own; no I/O except the shader-source lookups the recorder performs.

**Invariants** — the element index is the *deferred* element namespace: `0` high-quality (near — detail textures live), `1` low-quality (far), `2` shadow-map generation. An element the template does not answer emits no passes, and the draw stream then skips that object for that phase, which is the intended behaviour for, say, a forward-only material asked for its shadow pass.

```text
FUNCTION compile(C)
  base_compile(C)                     # sort priority + strict ordering knobs

  # 1. The forward detour
  forward = false
  IF use_alpha_channel AND alpha_ref < 16 THEN forward = true
  IF strict_back_front THEN forward = true

  IF forward THEN
    IF C.element IN {0, 1} THEN
      # one pass, source-alpha blend, alpha test at the authored reference,
      # depth-tested but NOT depth-writing, no fog
      C.pass(vertex="model_def_lq", pixel="model_def_lq",
             fog=true, depth_test=true, depth_write=false,
             blend=true, src=SRC_ALPHA, dst=INV_SRC_ALPHA,
             alpha_test=true, alpha_ref=alpha_ref)
      C.sampler("s_base", C.textures[0])
      C.end()
    # element 2 (shadow) emits nothing: a blended surface casts no shadow
    RETURN

  # 2. The deferred path
  aref = use_alpha_channel                 # here the flag means "alpha-test in the g-buffer pass"
  atoc = aref AND msaa_alphatest_mode IS alpha_to_coverage
  C.tessellation_method = tessellation     # honoured only where the device has tessellation

  IF C.element == NORMAL_HQ THEN emit_gbuffer(C, hq=true,  aref, atoc)
  ELSE IF C.element == NORMAL_LQ THEN emit_gbuffer(C, hq=false, aref, atoc)
  ELSE IF C.element == SHADOW THEN emit_shadow(C, aref)
```

**Notes** — The threshold `alpha_ref < 16` is the load-bearing constant. "Uses alpha" plus "cuts almost nothing out" means the artist wanted *fade*, not *cutout*; a cutout (glass frames, chain-link, foliage cards on a model) tests at a high reference and survives in the g-buffer as opaque-per-pixel. A rebuild that changes this number silently moves whole classes of model between the two lighting paths and changes how they look. `strict_back_front` forces the detour for the same reason, one level up: a material that asked for painter's-order sorting is by definition not resolvable by a depth buffer.

The three renderer generations in the source differ only in *how* a texture reaches a pass — a combined sampler versus a separate texture-and-sampler-object pair — and in whether the alpha-to-coverage and stencil-marking refinements exist. That split is an artifact of two graphics APIs, not a decision: a rebuild has one binding mechanism and one code path.

### `emit_gbuffer` — the g-buffer write

```text
FUNCTION emit_gbuffer(C, hq, aref, atoc)
  IF atoc THEN
    # A first pass writes ONLY coverage and stencil: colour writes are off and
    # alpha-to-coverage is on, so the multisample coverage mask is resolved from
    # alpha before any colour is committed.
    uber_deffer(C, hq, vertex_spec="model", pixel_spec="base_atoc", aref, finish=false)
    C.stencil(on, compare=ALWAYS, read_mask=0xff, write_mask=0x7f,
              fail=KEEP, pass=REPLACE, zfail=KEEP)
    C.stencil_ref(0x01)
    C.color_write(none)
    C.alpha_to_coverage(on)
    C.end()

  uber_deffer(C, hq, vertex_spec="model", pixel_spec="base", aref, finish=false)
  C.stencil(on, compare=ALWAYS, read_mask=0xff, write_mask=0x7f,
            fail=KEEP, pass=REPLACE, zfail=KEEP)
  C.stencil_ref(0x01)
  IF atoc THEN C.depth_compare(EQUAL)    # only the samples the coverage pass kept
  C.end()
```

**Invariants** — the stencil reference `0x01` written under mask `0x7f` is how the deferred lighting stage later knows "a real surface was written here". The mask deliberately leaves the top bit alone: that bit belongs to the multisample edge marker (see [`dx11MSAABlender.cpp`](dx11MSAABlender.cpp.md)), and a g-buffer write must not clear it.

### `emit_shadow` — the shadow-map write

```text
FUNCTION emit_shadow(C, aref)
  IF aref THEN
    # depth only, but the cutout must still cut: alpha test at a FIXED 220,
    # not at the authored reference
    C.pass(vertex="shadow_direct_model_aref", pixel="shadow_direct_base_aref",
           fog=false, depth_test=true, depth_write=true, blend=false,
           alpha_test=true, alpha_ref=220)
    C.sampler("s_base", C.textures[0])
  ELSE
    C.pass(vertex="shadow_direct_model", pixel=<none>,
           fog=false, depth_test=true, depth_write=true, blend=false)
  C.color_write(none)
  C.end()
```

**Notes** — The alpha reference in the shadow pass is hard-coded to **220**, not the authored value. A shadow map is rendered at a fraction of screen resolution, so a cutout that is generous at, say, 32 turns into a solid blob of shadow; raising the cut to 220 keeps only the strongly opaque texels and makes the shadow read as the silhouette an eye expects. This is a tuning constant with no derivation — reproduce it.

Where the device has no pixel shader requirement for a depth-only pass, the pixel stage is left empty; on devices that insist on one, a do-nothing pixel program named `dumb` fills the slot. That is a device requirement, not a decision.
