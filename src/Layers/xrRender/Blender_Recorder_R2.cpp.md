# src/Layers/xrRender/Blender_Recorder_R2.cpp

> The programmable half of the material compiler: a pass is a pair of named GPU programs, and every texture binding is resolved through the programs' own reflection data.

**Needs** — [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`Blender.h`](Blender.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`r_constants.h`](r_constants.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it compiles shader source, merges reflection tables and addresses sampler slots by device index.

## Purpose

The fixed-function recorder in [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) builds a pass by describing a combine chain. This one builds a pass by *naming two shader sources* and then asking the compiled result what it needs. That inversion is the whole difference between the renderer generations, and it is why the shipped shader source is part of the frozen data: the material system's texture bindings are resolved against identifiers that live in the shader files, not in the engine.

The split into its own file is arbitrary — it is the same class — and a rebuild should merge the two recorders behind one interface with two implementations, or keep one and generate shader source for the old path.

## `r_Pass(vertex, pixel, fog, depth_test, depth_write, blend, src, dst, alpha_test, alpha_ref)`

**Contract** — opens a programmable pass. Clears every accumulator, writes the fixed pipeline state the arguments ask for, compiles (or fetches from cache) the named programs, and merges each program's reflection table into the pass's constant table. Blocks on first use of a program while its source is compiled; subsequent uses hit the compiled-blob cache keyed by source hash and macro set. Does not append anything — `r_End` does that.

```text
FUNCTION r_pass(vs_name, ps_name, fog, ztest, zwrite, blend, src, dst, atest, aref)
  clear state_recorder, textures, matrices, constants, constant_table
  stage := 0

  set_depth(ztest, zwrite)
  set_blend(blend, src, dst, atest, aref)
  set_light_fog(light = false, fog)

  destination.pixel  := resources.compile_pixel_program(ps_name)
  constant_table.merge(destination.pixel.reflection)

  # A pixel program may declare that it needs the older constant-binding
  # semantics; that decision must reach the VERTEX compile, so the pixel
  # program is always compiled first and its flag fed forward.
  flags := destination.pixel.wants_legacy_binding ? legacy_binding : none
  destination.vertex := resources.compile_vertex_program(vs_name, flags)
  constant_table.merge(destination.vertex.reflection)

  destination.geometry := resources.compile_geometry_program("null")
  # ...and the remaining programmable stages are explicitly set to the
  # do-nothing program, never left absent (see Blender_Recorder.cpp)

  IF ps_name == "null" THEN
      # a pass with no pixel program still runs the fixed-function chain,
      # so its first stage must be explicitly terminated
      state_recorder.set_stage(0, colour_op, disable)
      state_recorder.set_stage(0, alpha_op,  disable)
```

**Invariants** — the ordering "pixel first, then vertex" is load-bearing, not stylistic: the legacy-binding flag is discovered by compiling the pixel program and must be applied to the vertex compile so that both halves of the pass agree on how constants are addressed. Compiling them in the other order gives a pass whose two programs disagree about register assignment.

**Notes** — On a device with linked program pipelines, the two programs are also linked into one object and *that* object's reflection is merged as well, because a linked pipeline may assign different locations than either program alone. A rebuild on an API where programs must be linked can skip the per-stage reflection entirely and read the linked program's.

## `r_End`

**Contract** — closes the pass: installs the standard constant binders, interns the state table, the constant table and the texture list, and appends the pass. Note what is *not* interned: the matrix and constant lists are left empty, because the programmable path has no fixed-function texture transforms or per-stage constants — everything that would have lived there is now a named shader constant with a binder.

## `r_Constant(name, binder)`

**Contract** — attaches a per-frame value producer to a named constant, if the pass's programs declare that name. Silently does nothing if they do not — which is the normal case, since the standard binding set installs dozens of names into every pass.

**Notes** — This is the *constant-table binding by name* that makes the shader data portable: the engine publishes a vocabulary of names (world matrix, fog parameters, sun direction, screen resolution…) and any shipped shader that spells one of them gets it filled every frame without the material knowing. The full vocabulary is in [`Blender_Recorder_StandartBinding.cpp`](Blender_Recorder_StandartBinding.cpp.md).

## `i_Sampler(name)`

**Contract** — resolves a sampler name against the pass's merged reflection table and returns its device slot index, or the invalid sentinel. The name is normalized first, the same way a texture name is. On a device that shares one namespace between samplers and textures the lookup is constrained to the sampler kind; otherwise the kind is left open and the caller separates them.

## The indexed sampler primitives — `i_Texture`, `i_Address`, `i_Projective`, `i_Filter*`, `i_BorderColor`, `i_Comparison`

**Contract** — each writes one sampler-state token for one already-resolved slot. `i_Filter` sets the minification, mip and magnification filters together, and on the OpenGL path additionally raises the anisotropy limit to the user's setting when both the min and mag filters asked for anisotropic — because that device expresses anisotropy as a *degree* on an otherwise-linear sampler rather than as a filter mode.

**Notes** — `i_Projective` writes a *texture-transform* token, not a sampler token, and only has an effect on slots below four. That is the fixed-function projective-divide flag surviving into the programmable path for the benefit of the oldest pixel-shader model, where projected texture lookups were a pipeline state rather than an instruction. On any modern rebuild it is dead and can be dropped — but the *call sites* must still pass the flag through, because the forward renderer's shadow projection relies on it.

## `r_Sampler(name, texture, projective, address, min, mip, mag)`

**Contract** — the workhorse. Resolves the sampler by name, binds the texture to it, then overrides the requested filtering for a handful of names before writing the sampler state. Returns the slot, or the invalid sentinel if the programs do not use the name.

```text
FUNCTION r_sampler(name, texture, projective, address, min, mip, mag)
  slot := i_sampler(name)
  IF slot invalid THEN RETURN invalid

  bind texture to slot        # by name on a separate-resource device;
                              # by the recursive resource path on a combined one

  # Name-keyed overrides. These exist so that fifty blenders need not each
  # remember which samplers deserve which filtering.
  IF name == "s_base"   AND min == linear THEN min, mag := anisotropic
  IF name == "s_detail" AND min == linear THEN min, mag := anisotropic
  IF name == "s_base_hud"                 THEN min, mag := the widest available
                                               reconstruction filter
  IF device is OpenGL THEN
      IF name == "s_position" THEN address := clamp ; min, mag := point ; mip := none
      IF name == "s_smap"     THEN address := clamp ;                     mip := none

  apply address, (min, mip, mag)
  IF slot < 4 THEN set projective division := projective
  RETURN slot
```

**Invariants** — the overrides only fire when the caller asked for the *default* linear filtering, so a blender that deliberately wants point filtering on the base texture still gets it.

**Notes** — Two of these are corrections rather than conventions and a rebuild should treat them as such. `s_position` is the deferred renderer's position/depth target: sampling it with any interpolation blends two surfaces' depths and produces a halo at every silhouette, so it must be point-sampled and clamped. `s_smap` is a shadow map, where a mip chain is meaningless and reading one produces shadow acne at distance. Both are guarded to the OpenGL path only because the Direct3D path sets them at the render-target's own sampler description instead; the *requirement* is device-independent.

`s_base_hud` asking for a wide reconstruction filter is the first-person weapon model's base texture, magnified hugely and always at the same screen depth; it is the one place the engine buys quality with a filter the rest of the frame cannot afford.

## `r_ColorWriteEnable(r, g, b, a)`

**Contract** — sets the colour write mask on **all four** render targets at once, not just the first. The deferred path binds several targets per pass and a material that means "write nothing" must mean it for every target; writing only target zero's mask leaves a depth-only pass scribbling into the normal buffer.

## The shorthands — `r_Sampler_rtf`, `r_Sampler_clf`, `r_Sampler_clw`

**Contract** — three named sampling conventions the shipped blenders reach for constantly:

- **rtf** — render target, full-screen: clamp, point, no mips. A texel-for-pixel blit.
- **clf** — clamped linear, no mips. A gradient or lookup ramp.
- **clw** — clamped linear on the first two axes, wrapping on the third. A 3D lookup table whose third axis is an angle or a time.
