# src/Layers/xrRender/blenders/Blender_Screen_SET.cpp

> The general-purpose material: everything about the pipeline state is a parameter, and the shipped data uses it for screen overlays, user-interface art, decals and anything else with no template of its own.

**Needs** — [`Blender_Screen_SET.h`](Blender_Screen_SET.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md) · [`HWCaps.h`](../HWCaps.h.md)
**Used by** — reached through its declarations in [`Blender_Screen_SET.h`](Blender_Screen_SET.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description; the parameter block is a frozen tagged byte stream.

## Purpose

Where every other template in this directory encodes *one kind of surface*, this one encodes none: it exposes the pass state directly as parameters and lets the material author assemble whatever they need. It is the escape hatch, and it is heavily used — the whole user interface, every full-screen effect overlay, and most wallmarks are materials of this template.

One implementation serves every renderer generation; the branch inside is on whether the device has a fixed-function pipeline at all, not on which renderer is running.

## State

```text
RECORD Parameters
  blend_mode : enum, one of 10, default 0   # index order frozen; see below
  clamp      : bool, default true
  alpha_ref  : int in [0,255], default 32
  depth_test : bool, default false
  depth_write: bool, default false
  lighting   : bool, default false
  fog        : bool, default false
```

**The ten modes, in their frozen order.** The first six are the same six the particle template uses, by index and by meaning; the last four were added later.

```text
0  SET              replace, no alpha test
1  BLEND            src alpha : inv src alpha, alpha test at alpha_ref
2  ADD              one : one
3  MUL              dest colour : zero
4  MUL_2X           dest colour : src colour
5  ALPHA-ADD        src alpha : one, alpha test at alpha_ref
6  MUL_2X (B^D)     as 4, but composed differently — the wallmark mode
7  SET (2r)         replace, with the texture read at DOUBLE intensity
8  BLEND (2r)       as 1, with the texture read at double intensity
9  BLEND (4r)       as 1, with the texture read at QUADRUPLE intensity
```

**Invariants** — the "(2r)" and "(4r)" suffixes mean *range*, not blending: those modes multiply the sampled texture by 2 or by 4 before anything else, so that a texture authored in the 0..1 range can drive a high-dynamic-range overlay. That multiplication happens in the combine chain or in the pixel program, never in the frame blend, and it is the only difference between modes 1, 8 and 9.

## `Save` / `Load`

**Contract** — writes the blend selector with all ten labels inlined, then the clamp flag, the alpha reference, and the four state booleans. Reading is version-gated in one place: parameter version 2 has no clamp flag and the rest shift up by one field. In both cases the option count is **forced to ten after reading**, which is how a record written when there were seven modes still selects correctly among ten.

**Notes** — The constructor sets the version to 4 and the option count to 9, while the loader forces 10. The three numbers disagree and the discrepancy is not explained anywhere: the safe reading is that the count is presentation-only (it bounds the authoring tool's menu) and that forcing it to the current maximum on load is the intended behaviour, with the constructor's value simply stale. What a rebuild must preserve is the *index* meaning of all ten, not the count.

## `Compile`

**Contract** — one pass, always. The texture binding and the intensity multiplier come from one of two branches; the state comes from the parameters.

```text
FUNCTION compile(context)
  pass:
    IF the device has a fixed-function pipeline THEN emit_fixed() ELSE emit_programmable()

    set lighting and fog from the parameters
    set depth test and depth write from the parameters

    SELECT blend_mode
      SET           -> no blending, no alpha test
      BLEND         -> src alpha : inv src alpha, alpha test at alpha_ref
      ADD           -> one : one,                 no alpha test
      MUL           -> dest colour : zero,        no alpha test
      MUL_2X        -> dest colour : src colour,  no alpha test
      MUL_2X (B^D)  -> dest colour : src colour,  no alpha test
      SET (2r)      -> one : zero,                alpha test at 0
      BLEND (2r)    -> src alpha : inv src alpha, alpha test at alpha_ref
      BLEND (4r)    -> src alpha : inv src alpha, alpha test at alpha_ref
```

### The fixed-function emission

```text
FUNCTION emit_fixed()
  IF blend_mode is MUL_2X (B^D) THEN
      # the wallmark shape: the decal's texture, then the vertex colour
      # blended over it by the VERTEX colour's own alpha
      stage 0: base texture selected, clamped if the flag is set
      stage 1: vertex colour blended over the running value by the vertex alpha;
               alpha := vertex alpha * running alpha
  ELSE
      stage 0: base texture times vertex colour, at 1x, 2x or 4x
               depending on the mode, clamped if the flag is set,
               sampled through the template's base texture and matrix
```

### The programmable emission

```text
FUNCTION emit_programmable()
  SELECT blend_mode
    MUL_2X (B^D)           -> programs "stub_notransform_t"    / "stub_default_ma"
    BLEND (4r)             -> programs "stub_notransform_t_m4" / "stub_default"
    SET (2r), BLEND (2r)   -> programs "stub_notransform_t_m2" / "stub_default"
    everything else        -> programs "stub_notransform_t"    / "stub_default"

  bind s_base <- instance texture 0 through the base sampler
  IF clamp THEN force clamp addressing on the resolved sampler
  # at least one texture must be named, or this is an authoring error
```

**Invariants** — the intensity multiplier is encoded in the **vertex program's name suffix** (`_m2`, `_m4`), not in a constant. That is why the three blend modes that differ only in range need three program pairs, and it is why those exact program names are part of the frozen shader-data contract.

**Notes** — Mode 6's name says it is a doubling multiply, and its frame blend is exactly mode 4's. What distinguishes it is the *composition before the blend*: it uses the vertex colour's alpha as the mix weight between the texture and the vertex colour, which is how a wallmark fades out over its lifetime — the decal's age is written into the vertex alpha by the decal store, and the material reads it here. Its programmable pixel program is the only one in the set with a distinct name, `stub_default_ma`, for that reason.
