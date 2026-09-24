# `src/Layers/xrRenderPC_GL` — registering the OpenGL renderer, and composing its frame

Chapter 21's other half, with [`../xrRenderGL`](../xrRenderGL/README.md). See
[the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

This is the module the engine actually loads. It does four jobs, and only the first is
administrative:

1. **It announces the renderer.** One exported entry point hands the engine a module object;
   the module publishes the renderer names and identifiers it can answer to, probes whether
   the machine can run it at all, checks that the game data contains what it needs, and — when
   chosen — fills the global environment struct of
   [chapter 5](../xrAPI/README.md) with itself.
2. **It compiles the shipped shader sources** into device programs, which means deciding the
   macro set, resolving textual inclusion by hand because the shading language has no such
   facility, and maintaining the on-disk program cache.
3. **It builds the deferred frame's render targets** and the procedural lookup textures the
   lighting resolve reads.
4. **It supplies the parts of the frame graph whose target juggling is backend-specific** —
   the final combine and post chain, the sun accumulation, and the sun's own shadow phase.

The companion module for the Direct3D 11 filling is
[`../xrRenderPC_R4`](../xrRenderPC_R4/README.md) and it does the same four jobs; reading the
two side by side is the cheapest way to see which parts of the seam are real and which are
archaeology.

## Where it sits

It rests on [chapter 18](../xrRender/README.md) for the renderer core, on
[chapter 19](../xrRender_R2/README.md) for the frame graph it is composing, and on
[`../xrRenderGL`](../xrRenderGL/README.md) for every device call it makes — that is where the
translations named in this chapter's first half are performed. It reaches the engine for the
device, the console and the global environment struct, and it reaches the virtual filesystem
for two corpora of shipped data: the shader sources, and the cached compiled programs it
writes back beside the user's settings.

It is the last link in the chain: nothing depends on it except the executable
([chapter 27](../../xr_3da/README.md)), which holds a fixed list of renderer modules and asks
each in turn what it can offer.

---

## The ideas you need before the twins make sense

### A renderer is chosen by *name*, and the name is shipped data

The seam requires that a renderer be selectable at startup by name and that a failed device
creation fall back to the next candidate. The names are not this module's to invent — the
game's own configuration and presets spell them, so they are frozen. There are five in
circulation, one per generation of the original engine's renderers, and this module publishes
either of two:

- On Windows it publishes **one name of its own**, with the next free identifier after the
  shipped set. There is a Direct3D backend on that platform, so the OpenGL renderer is an
  additional, explicitly chosen option.
- On every other platform it publishes **the name of the Direct3D 10-class renderer** — because
  no Direct3D backend exists there, and the shipped configuration, presets and menu still spell
  that name. The OpenGL filling answers to it. This is the single decision that makes the
  engine portable without touching a line of shipped data, and it is worth copying.

Whichever name is chosen, the module then forces the same two quality decisions — the static
sun is off, the advanced post chain is on — and declares a generation identifier the *game*
branches on. It claims the Direct3D 10-class level, not the 11-class one.

### Advertisement is binary here, graded there

The other filling's probe returns three answers — cannot run, can run at the 10-class level,
can run at the 11-class level — and its module turns that grading into a list of up to four
selectable names, so one binary serves a decade of hardware.

This module's probe answers **yes or no**, and the way it answers is worth naming because it is
the honest way: it does not query extensions or limits. It **creates a one-pixel hidden window,
brings a real device up on it, and asks whether a context and a framebuffer came back** — then
tears the whole thing down. Capability probing by construction. A rebuilder with a
device-creation path that is cheap and side-effect-free should prefer this to a feature matrix,
because it tests the thing that actually has to work.

There is a second gate, and it is about data rather than hardware: **the renderer refuses to
offer itself if its shader directory is absent from the game data.** That is the seam declining
to load rather than failing later with a wall of compile errors, and it is the mechanism that
makes the next idea safe.

### The shipped shader corpus is a *second* corpus, translated ahead of time

[§5](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) names shader source shipped as game
data, in a dialect written for another API, as the single largest compatibility hazard in a
port. The answer here is the one a rebuilder should weigh first: **translate once, at authoring
time, and ship the result.** This backend reads its programs from a different directory of the
game data than the Direct3D backend does, in this API's own shading language, maintained by the
project. There is no run-time dialect translator anywhere in the engine.

What *does* happen at load, per stage, is a smaller and much more tractable job:

- **A prelude is prepended**: the language version, the extension the separable-program path
  needs, an optimization pragma, and a comment naming the source — the comment is not decoration,
  it is what makes a driver's error messages legible.
- **The option macros are defined.** Roughly forty of them, and they are the same forty the other
  filling defines, because they are the shipped corpus's own vocabulary: shadow-map size, the
  half-precision filter and blend capabilities, hardware shadow lookup and its filtered variant,
  jitter, branching, vertex texture fetch, translucent shadows, motion blur, the sun filter and
  the static-sun mode, gloss forcing, skin colouring, the ambient-occlusion family and its
  quality steps, the skinning formulation, soft water, soft particles, screen-space reflections
  and their quality steps, depth of field, sun-shaft quality, sun quality, steep parallax, the
  g-buffer-optimized layout, the shader-model flag, the min/max shadow map, the older games'
  resource set, and the multisampling family. **The set of macro names is part of the frozen
  contract** — the shipped sources test them by name.
- **Textual inclusion is resolved by hand**, because this shading language has none. Each
  inclusion directive is found, the named file is read from the virtual filesystem with its
  separators normalized, and its text is spliced into the source list; the splice recurses. One
  known limitation: a commented-out inclusion is still honoured.
- **One macro is suppressed on one platform.** The shader-model flag is not defined on macOS,
  because that platform's implementation of this language version requires a compile-time
  constant where the guarded code passes a computed one. A rebuild targeting several drivers
  will accumulate its own list of these, and the lesson is that the list exists at all.

The **program cache** is what makes a warm start affordable, and its key is the interesting
part. A cache entry is stored under the shader's name and stage, and its header records the
adapter name, the API version string and the shading-language version string — so a driver
update invalidates every entry — followed by the binary format identifier, a checksum of the
source **together with all of its transitive inclusions**, and a checksum of the binary itself.
Including the inclusions in the key is the non-obvious half: the shared headers of the corpus
are what change most often. The cache is used only when the driver both supports retrievable
program binaries and supports separable programs; the monolithic-link path has no cache and
recompiles from source every start.

### What is composed here, and why it could not stay in chapter 19

Chapter 19 owns the frame graph and says so; this module owns the handful of steps where *which
surface is bound to what* differs between the two fillings. Three things collect here:

- **The target set and the lookup textures.** Every deferred target is declared here because the
  handle type and the creation call are the backend's. The procedural lookup textures — the
  three-dimensional lighting table indexed by the two half-angle cosines and the material row,
  and the set of jitter textures the soft-shadow and ambient-occlusion passes sample — are built
  by generating the same numbers and uploading them into immutable storage.
- **Binding a phase's targets.** Because this API's "render targets" are attachments on one
  long-lived framebuffer rather than a bound array, every phase's target binding must also
  declare *which attachments the fragment stage writes*, and must explicitly attach nothing
  where the other filling would simply pass a shorter list. Each such binding ends with a
  framebuffer-completeness check, which has no counterpart on the other filling.
- **The final combine, the sun accumulation and the legacy sun phase.** These carry the
  texture-origin and depth-range compensations named in
  [the first half of this chapter](../xrRenderGL/README.md) — the shadow texgen matrix's halved
  and biased depth row, the reordered full-screen quads, the inverted texture coordinates — and
  the legacy sun phase carries the largest one of all: it is rewritten against a vector-math
  library whose default clip-depth convention is this API's, so the substitution of the math
  layer *is* the compensation.

### Where this filling is thinner than its sibling, honestly

A rebuilder deserves the list, because it is what a port costs when it is not finished:

- **Multisampling is off**, so chapter 18's stencil-bit edge scheme and chapter 19's per-sample
  lighting never run. Where the shared frame graph would enter a per-sample path, this filling
  refuses outright rather than approximating.
- **The compute-shader ambient-occlusion paths and the horizon-based variant are disabled**, and
  the combine phase's branch for them is empty.
- **Volumetric fog is off.**
- **Render-target arrays are unsupported**, so the sun cannot render its cascades into slices of
  one array target. That, and not style, is why this module carries its own copy of the legacy
  sun phase; and it is why the sun's near and far passes share a single accumulation slot rather
  than one per cascade.
- **The stencil-bounds optimization is unimplemented** and refuses if reached.
- **There is no asynchronous screenshot surface**, so the save-thumbnail and tooling captures
  have no body.
- **The final flip phase is dead.** It survives only as a readable record of the full-screen
  textured quad — half-pixel offset and all — that the other API needed. Presentation here is a
  framebuffer-to-framebuffer blit with no flip, done by the device itself, because the whole
  pipeline already lives in this API's coordinate space. That deletion is a small, exact example
  of the chapter's thesis: the quad existed because of a convention, not because of the hardware.

---

## Files

Build files are omitted; they describe how the original was compiled, not what it does.

| File | Role |
|---|---|
| [`xrRender_GL.cpp`](xrRender_GL.cpp.md) | **The registration**: the one exported entry point, the renderer names and identifiers this backend answers to, the platform rule that decides which, the data-presence gate, and the installation of the renderer into the global environment struct |
| [`r2_test_hw.cpp`](r2_test_hw.cpp.md) | The capability probe, answered by building a throwaway device on a one-pixel hidden window and asking whether it came up |
| [`rgl_shaders.cpp`](rgl_shaders.cpp.md) | **Compiling a shipped shader source**: the injected prelude, the frozen option-macro set, hand-resolved textual inclusion, and the program cache keyed by driver identity plus a checksum over the source and every file it includes |
| [`gl_rendertarget.h`](gl_rendertarget.h.md) | Declares the whole deferred target set, the pass descriptions and the post-process parameters for this filling — the place where every target handle's type is fixed |
| [`gl_rendertarget_build_textures.cpp`](gl_rendertarget_build_textures.cpp.md) | Generating the lighting lookup volume and the jitter texture set, and uploading them into immutable storage |
| [`gl_rendertarget_u_set_rt.cpp`](gl_rendertarget_u_set_rt.cpp.md) | Binding a phase's targets: attachments on the one owned framebuffer, the explicit declaration of which attachments are written, and the completeness check that follows |
| [`gl_rendertarget_accum_direct.cpp`](gl_rendertarget_accum_direct.cpp.md) | **Sun accumulation**: the stencil mask, the shadow texgen matrix carrying both the texture-origin and the clip-depth compensation, the cascade volume draw, and the light-shaft pass |
| [`gl_rendertarget_phase_combine.cpp`](gl_rendertarget_phase_combine.cpp.md) | The final composition: sky, the tone-mapped lighting resolve, the forward pass, volumetric compositing, bloom, distortion, and the anti-aliasing, depth-of-field and motion-blur resolve |
| [`r2_R_sun.cpp`](r2_R_sun.cpp.md) | The legacy sun phase for this filling — the caster hull, the trapezoidal light matrix and the shadow-map render, rebuilt on a math layer whose depth convention is this API's |
| [`gl_rendertarget_phase_flip.cpp`](gl_rendertarget_phase_flip.cpp.md) | The retired final blit, kept as a record of the full-screen quad and half-pixel offset that presentation no longer needs |
| [`stdafx.h`](stdafx.h.md) | Fixes this module's renderer identity for the shared sources it compiles, and holds the one shared helper that binds the four jitter samplers |
| [`stdafx.cpp`](stdafx.cpp.md) | A compilation artifact with no content of its own |
