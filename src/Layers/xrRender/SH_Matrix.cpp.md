# src/Layers/xrRender/SH_Matrix.cpp

> An animated texture-coordinate matrix: the four ways the shipped materials move a texture across a surface — scroll, rotate, scale, and the two camera-derived reflection projections.

**Needs** — [`SH_Matrix.h`](SH_Matrix.h.md) · [`xrEngine/WaveForm.h`](../../xrEngine/WaveForm.h.md)
**Used by** — [`SH_Matrix.h`](SH_Matrix.h.md)
**Tier floor** — T1: its five waveforms and its mode word are read from the shipped material library as a byte image.

## Purpose

Water scrolls, a scanner sweeps, a chrome surface reflects the room. Before programmable shading those were all expressed as a matrix applied to the texture coordinates, and the shipped material library still describes them that way. This record is that matrix and the schedule that drives it.

## State

```text
RECORD Matrix
  name       : text
  xform      : matrix (4x4)         # the computed result
  frame      : int                  # once-per-frame memo, exactly as for a constant
  mode       : enum { programmable, tcm, spherical_reflection, cubic_reflection, detail }
  tcm_flags  : bit set { scale, rotate, scroll }
  scaleU, scaleV, rotate, scrollU, scrollV : WaveForm
```

**Invariants**

- The flags are meaningful only in the `tcm` mode; the other modes ignore them entirely.
- In `programmable` and `detail` modes nothing is computed — the matrix is whatever was assigned. Detail texturing derives its matrix from the surface's detail scale elsewhere, and the mode exists here only so the compiler can mark that the matrix is somebody else's business.
- The waveforms produce a *rate*, not a value, for rotation and scrolling: both are multiplied by the current time after evaluation, so a constant waveform yields a linear sweep. Scaling is used directly. The asymmetry is deliberate and frozen — the shipped parameters are authored as speeds.

## `Calculate`

**Contract** — recompute the matrix for the current frame, at most once, according to the mode. Reads global time and, in the two reflection modes, the current view matrix. Cheap; called from the bind path.

### `tcm` — scroll, rotate and scale about the texture's centre

```text
FUNCTION calculate_tcm()
  t = global time
  xform = translate(+0.5, +0.5)              # move the texture's centre to the origin
  IF rotate flag
    xform = xform then rotate_about_z(rotate.at(t) * t)
  IF scale flag
    sU = scaleU.at(t) ; sV = scaleV.at(t)
    xform = xform then scale(sU, sV)
  IF scroll flag
    u = scrollU.at(t) * t * sU               # scroll is expressed in pre-scale units
    v = scrollV.at(t) * t * sV
    xform = xform then translate(u, v)
  xform = translate(-0.5, -0.5) then xform   # move the centre back
```

**Notes**

- The half-unit bracket is what makes rotation and scaling happen about the middle of the texture rather than its corner. It is a frozen convention: the shipped parameters were authored against it.
- Scroll is multiplied by the scale factors, so a texture that has been scaled up scrolls proportionally faster and the apparent speed across the *surface* stays constant. When the scale flag is clear the factors are 1, so the multiplication is harmless. This coupling is the non-obvious decision in the function.
- A texture-coordinate translation is written into the third row of the matrix, not the fourth: texture coordinates are a two-component vector extended with a 1 in the third slot, so the translation lives where a 3-vector transform would put it. A rebuild using 3x2 texture matrices places it differently and must be careful not to transcribe the row index.

### `spherical_reflection` — environment mapping from the view matrix

Builds a matrix that maps a world-space position to a texture coordinate by taking the view matrix's first two basis rows, halving them, negating the second, and biasing both by a half. The halve-and-bias is the standard mapping from the `[-1,1]` clip range into the `[0,1]` texture range; the negation flips the vertical axis because texture space runs downward. The third and fourth columns are zeroed — the result is a 2-D coordinate and the remaining components must not leak.

### `cubic_reflection` — the view rotation, inverted

Takes the view matrix, drops its translation, and inverts what remains. The result maps a view-space direction back to a world-space direction, which is exactly what a cube map wants to be indexed by. Dropping the translation before inverting is what makes it a rotation-only inverse; a rebuild that inverts the full matrix gets a different and wrong answer.

## `Load` · `Save`

**Contract** — mode word, flag word, then the five waveforms as byte images in the order scaleU, scaleV, rotate, scrollU, scrollV. **Frozen.** Unlike the colour constant, the mode *is* stored here, because a matrix's mode is a genuine authoring choice.

## `Similar`

**Contract** — true when two matrices animate identically: same mode, same flags, and five channel-wise similar waveforms. Used to share one record between passes.

## `tc_trans`

**Contract** — builds the texture-coordinate translation matrix described above. A named step of the `tcm` computation rather than an independent unit.
