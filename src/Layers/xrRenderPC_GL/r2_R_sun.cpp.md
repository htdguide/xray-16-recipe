# src/Layers/xrRenderPC_GL/r2_R_sun.cpp

> The OpenGL build's own copy of the legacy two-region sun, substituted for the shared one
> at build time; same algorithm, different matrix library, and a handful of divergences
> that are not all deliberate.

**Needs** — [`../xrRender_R2/render_phase_sun_old.cpp`](../xrRender_R2/render_phase_sun_old.cpp.md) ·
[`../xrRender_R2/r2_R_sun_support.h`](../xrRender_R2/r2_R_sun_support.h.md) ·
[`../xrRender_R2/r2.h`](../xrRender_R2/r2.h.md) ·
[`../xrRender_R2/r2_types.h`](../xrRender_R2/r2_types.h.md) ·
[`../xrRender_R2/r3_rendertarget_phase_smap_D.cpp`](../xrRender_R2/r3_rendertarget_phase_smap_D.cpp.md) ·
[`../xrRender_R2/r2_rendertarget_phase_accumulator.cpp`](../xrRender_R2/r2_rendertarget_phase_accumulator.cpp.md) ·
[`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md) ·
[`../xrRender/light.h`](../xrRender/light.h.md) ·
[`../xrRender/r__sector.h`](../xrRender/r__sector.h.md) ·
[Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md)
**Tier floor** — T1: projective matrix construction against a specific clip-space
convention, and direct submission to the immediate command list.

## Purpose

Two sun implementations ship — the cascaded one and the legacy trapezoidal one — and the
R2 chapter owns both. This backend compiles the shared **cascaded** sun unchanged, but
replaces the shared **legacy** sun with this file. So the legacy sun exists twice in the
tree, and which copy a build gets is decided by which source list it compiles, not by any
runtime choice.

The algorithm is not restated here: it is in
[`../xrRender_R2/render_phase_sun_old.cpp`](../xrRender_R2/render_phase_sun_old.cpp.md) —
the snapped near region, the trapezoidal warp of the far region, the focus refit, the
guaranteed range. This page records what is different in this copy.

The honest framing for a rebuilder: **this file should not exist.** Nothing in it is an
OpenGL design decision. It is a port that was made by rewriting every matrix expression
against a portable matrix library, and the differences below are what that rewrite left
behind. A rebuild has one legacy sun, parameterized by its clip-space convention, and
deletes this page.

## State

The same phase record as the shared legacy sun, with one change: the light's shadow
transform array is used differently.

```text
# The shared implementation stores each region's transform in the array slot named
# by that region's sub-phase: near in slot 0, far in slot 2.
# This copy stores BOTH regions in slot 0, overwriting between them.
```

**Invariants** — slot 0 holds *whichever region is currently live*, and the backend's sun
accumulation on this filling reads slot 0 to match (see
[`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md)). The pairing is
consistent but it is load-bearing: it only works because the near region is rendered *and
accumulated* before the far region is fitted. The shared implementation's per-sub-phase
indexing does not depend on that ordering, and is the form a rebuild should take.

**Notes** — the shared implementation also selects the shadow array's read slice per
sub-phase before accumulating; this copy does not. On this backend that call is a no-op —
the whole array is bound and the program indexes it — so the omission is harmless here and
would not be elsewhere.

## `initialise`

**Contract** — as the shared implementation's, with one change: **parallel visibility
calculation is permitted**, taken from the renderer's option, where the shared
implementation forces it off.

**Notes** — the shared implementation disables it because the legacy sun's far region
consumes the receiver boxes the main view's walk records, which serialises the two walks.
That constraint is a property of the algorithm, not of the API, so enabling parallelism
here is a difference this page can record but not justify: **the reason is not
recoverable**, and a rebuild should follow the shared implementation and keep it off.

## `render`

**Contract** — as the shared implementation's: near region, then far region, then the
optional filtered resolve. Near before far remains required, and here it is required
twice over — for the shared reason (both regions write the same shadow map) and for this
copy's own (both regions write the same transform slot).

## `render_near` / `render_far`

**Contract** — as the shared implementation's, fitted and drawn identically. Two
differences in how the result is issued, and two in how it is computed.

**How it is issued.** Every draw goes to the **immediate command list**, not to the
phase's own context. The context is still allocated, still used for the visibility walk,
and still submitted and released in `flush` — but it records no draws. The consequence is
that on this filling the legacy sun can overlap its *visibility* with other work and never
its *drawing*, whatever the parallel-render option says. The shared implementation records
into the phase's context throughout and can do both.

**Invariants** — because the draws bypass the phase's context, the camera transforms this
file restores at the end of each region are restored on the immediate context, which is
where the next pass expects them. A rebuild that moves these draws back onto the phase's
context must move the restore with them or the pass after the sun inherits the light's
projection.

**How it is computed.** Two constructions in the far region's trapezoidal warp differ from
the shared implementation in ways that change the matrix, not just its spelling:

- The final light-view-projection chain is built as *light basis × orthographic ×
  trapezoid*, **omitting the camera view matrix** that the shared implementation multiplies
  in first — and it is accumulated into a matrix that was never initialised.
- The light direction is carried into eye space by composing a translation with the view
  matrix and evaluating it at a fixed point, rather than by rotating a direction. A
  direction and a position do not transform the same way.

Both are inside the branch that runs only when the trapezoidal warp is enabled *and* the
sun is not nearly parallel to the view. Whether this backend ships with the warp disabled —
which would make both dead — or whether its far shadows are simply wrong is **not
recoverable** from the source: no comment, no guard and no alternative path records an
intent. A rebuild should take the shared implementation's construction as the specification
and treat these two as transcription errors.

**A third difference is more likely deliberate but still unexplained.** The near region's
viewport matrix — the one used only to convert the fitted hull into the scissor rectangle
the accumulation later reads — negates its vertical axis, which is the *other* API's window
origin. Everywhere else in this backend that negation is conditioned on the API (compare
the rain map's viewport matrix in
[`../xrRender_R2/r3_R_rain.cpp`](../xrRender_R2/r3_R_rain.cpp.md), which flips it). Left as
it is, the near region's scissor is vertically mirrored. It is not used to set a viewport,
only to build the shadow lookup's sub-rectangle, which is why the error is survivable.

## `render_filtered`

**Contract** — as the shared implementation's: when the sun-filter option is on, binds the
accumulator and runs one more sun accumulation in the luminance sub-phase. Issued on the
immediate command list here, for the same reason as the regions.

## `flush`

**Contract** — submits and releases the phase's context and invalidates the immediate
command list. The submission is vacuous — the context recorded no draws — but the release
is not: it returns the context to the fixed pool that bounds how many phases may be in
flight.
