# src/Layers/xrRender/blenders/Blender_Screen_GRAY.cpp

> Desaturation as a full-screen pass, computed on fixed-function hardware with a dot product against a luminance constant.

**Needs** — [`Blender_Screen_GRAY.h`](Blender_Screen_GRAY.h.md) · [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md) · [`Blender_CLSID.h`](../Blender_CLSID.h.md)
**Used by** — reached through its declarations in [`Blender_Screen_GRAY.h`](Blender_Screen_GRAY.h.md); callers name that, not this file.
**Tier floor** — T2: it emits a pass description with no parameter block.

## Purpose

An internal template the engine constructs directly — the grey-out used for death, unconsciousness and the pause overlay. Forward renderer only.

The whole file is one idea: how to compute a weighted luminance when the only tool available is a three-component dot product on unsigned 8-bit values.

## `Compile`

```text
FUNCTION compile(context)
  pass:
    depth test and write off, blend replace, lighting and fog off
    the pass's shared constant colour := (181, 255, 134, 0)

    stage 0: base texture ADDED to vertex colour, clamped addressing,
             sampled through the template's base texture and matrix
    stage 1: the running value DOT-PRODUCT-3 the pass constant,
             same texture and matrix, both colour and alpha channels
```

**Invariants** — the constant is the standard luminance weights **biased by 105 on each of the three colour channels**. The weights themselves are the usual ones scaled to 8 bits: red 76, green 150, blue 29. The bias exists because the fixed-function three-component dot product treats its inputs as signed values biased at 128 — it computes `4 * sum((a - 0.5) * (b - 0.5))` — so a weight that should be a small positive number has to be offset above the midpoint to survive the bias. 105 is not derivable from the weights; it is the offset that makes the biased product land where the unbiased one would. **This constant is only correct for that exact dot-product convention**; a rebuild computing luminance in a shader uses the three unbiased weights and no offset.

**Notes** — Stage 0 *adds* the vertex colour rather than modulating by it, which lets the caller brighten or tint the image before it is desaturated by varying the quad's vertex colour. The alpha channel of the constant is zero, so alpha is carried through the same dot product and comes out near the midpoint — nothing reads it, because the pass replaces the frame outright.
